# Terminal Performance Optimization

## Context

Les terminaux de Dorothy sont moins fluides que ceux de WaveTerm (même stack xterm.js). Trois bottlenecks identifiés :
1. **Sérialisation `Vec<u8>` → JSON array** : chaque chunk PTY de 4KB devient ~12KB de JSON `[72, 101, ...]` — inflation 3x
2. **Watermarks trop élevés** : 500KB avant pause → xterm.js doit digérer un gros batch d'un coup → frame drops
3. **Polling 10ms dans le reader thread** : `thread::sleep(10ms)` en boucle au lieu d'un signal propre → latence + CPU gaspillé

## Plan d'implémentation

### Phase 1 — Encodage base64 des données PTY (impact critique)

> **Note** : `serde_bytes` ne fonctionne PAS avec `serde_json` — il sérialise toujours en array. Il faut encoder en base64 manuellement.

**Fichiers Rust :**
- `src-tauri/Cargo.toml` — ajouter `base64 = "0.22"`
- `src-tauri/src/pty.rs` (lignes 18-23) — encoder `data` en base64 string avant émission :
  ```rust
  use base64::Engine;
  // ...
  pub struct PtyOutputEvent {
      pub agent_id: String,
      pub pty_id: String,
      pub data: String,  // base64-encoded bytes
  }
  // Dans le reader thread (ligne 154-158):
  let b64 = base64::engine::general_purpose::STANDARD.encode(data);
  let event = PtyOutputEvent {
      agent_id: agent_id_owned.clone(),
      pty_id: pty_id_owned.clone(),
      data: b64,
  };
  ```

**Fichier utilitaire frontend :**
- `src/lib/base64.ts` (nouveau) — helper de décodage :
  ```typescript
  export function decodeBase64(b64: string): Uint8Array {
    const bin = atob(b64);
    const bytes = new Uint8Array(bin.length);
    for (let i = 0; i < bin.length; i++) bytes[i] = bin.charCodeAt(i);
    return bytes;
  }
  ```

**14 fichiers frontend** — changer `data: number[]` → `data: string` et utiliser `decodeBase64()` :
- `src/lib/terminal-write.ts` — pas de listener direct, mais le type de `write()` reste `Uint8Array` (OK)
- `src/hooks/useAgentTerminal.ts`
- `src/hooks/useElectron.ts`
- `src/components/TerminalsView/hooks/useMultiTerminal.ts`
- `src/components/Terminal.tsx`
- `src/components/MosaicTerminalView/TerminalTile.tsx`
- `src/components/TrayPanel/useTrayTerminal.ts`
- `src/components/AgentTerminalDialog/useQuickTerminal.ts`
- `src/routes/console.tsx`
- `src/routes/ssh-terminal.tsx`
- `src/routes/docker.tsx`
- `src/components/TerminalDialog.tsx`
- `src/components/Settings/InstallTerminalModal.tsx`
- `src/components/NewChatModal/hooks/useSkillInstall.ts`

**Gain** : payload 4KB → ~5.3KB (base64) au lieu de ~12KB (JSON array). Réduction ~55%.

---

### Phase 2 — Réduire les watermarks (trivial)

**Fichier** : `src/lib/terminal-write.ts` (lignes 4-5)
```typescript
const HIGH_WATERMARK = 128 * 1024; // 128KB (était 500KB)
const LOW_WATERMARK = 32 * 1024;   // 32KB (était 100KB)
```

**Gain** : backpressure déclenché plus tôt → moins de frame drops sur les bursts.

---

### Phase 3 — Remplacer le polling par un Condvar (propre)

**Fichier** : `src-tauri/src/pty.rs`

- Ajouter `pause_signal: Arc<(std::sync::Mutex<bool>, std::sync::Condvar)>` à `PtyHandle`
- Reader thread : remplacer la boucle `while paused + sleep(10ms)` par `cvar.wait_timeout(5s)`
- `resume()` : appeler `cvar.notify_one()` en plus du `store(false)`
- `pause()` : mettre le mutex à `true`

**Gain** : 0 CPU quand pausé, reprise instantanée (vs 0-10ms de latence aléatoire).

---

### Phase 4 — Upgrade xterm.js v5 → v6 (séparé, scope large)

**Pas dans ce commit.** La migration v6 touche 23 fichiers avec des renommages de packages (`xterm` → `@xterm/xterm`, etc.) et des changements d'API potentiels. À faire dans un PR dédié après les 3 premières phases.

---

## Vérification

1. `cargo build` — vérifier que le backend compile
2. `pnpm dev` / `pnpm tauri dev` — lancer l'app
3. Ouvrir un terminal → taper des commandes → vérifier la réactivité
4. Tester un burst : `cat /dev/urandom | head -c 100000 | base64` → vérifier pas de freeze
5. Ouvrir un agent terminal (Path B WebSocket) → vérifier que rien n'a régressé
6. Vérifier le terminal SSH, Docker, console, mosaic, tray, quick terminal
