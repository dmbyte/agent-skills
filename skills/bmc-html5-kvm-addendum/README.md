# Proposed changes to the installed `bmc-html5-kvm` skill

The shipped skill states "Initial target is HPE iLO 5" and "an actual iLO 5
console has not yet been validated". On iLO4 v2.82 the documented assumptions
below were wrong or missing. All of it was validated on a DL360p Gen8.

## Add

1. **iLO4 is a frameset.** `inspect()` returns `controls: []` unless
   `select_frame(n)` was called for the frame that owns the control, and frame
   indexes shift as iLO loads (login page, application, masthead, navigation,
   content). Re-select and re-inspect after every navigation instead of caching
   selectors or indexes.
2. **256-character selector limit is hit constantly on iLO4.** Its generated CSS
   paths are 200-280 characters. iLO4 exposes good ids - use them:
   `#usernameInput`, `#passwordInput`, `#ID_LOGON`, `#toggleExpandCollapseAll`,
   `#tabset_rc`, `#HRCButton`, `#IRCButton`, console surface `#rc_video`.
3. **Console focus is lost on server power events and after iLO logouts.**
   `focus_console` on a page that has been logged out raises
   "Select exactly one visible console canvas ... may only be a management
   page"; the recovery is the full re-login + relaunch flow, not a retry.
4. **Keystroke fidelity is guest-dependent, not just "input requires focus".**
   Reliable through POST/RBSU/GRUB; under Linux the same calls drop keys,
   duplicate them and lose Shift, and delivery is bursty (characters surface
   seconds later). Add guidance: never type credentials, installer answers or
   shell commands through this channel; use the BMC serial console (SOL) or a
   boot-time payload instead, and verify every firmware string by capture
   before confirming it (Enter / F10).
5. **A blank or unchanged capture is ambiguous.** Document a pixel-diff check
   so operators can tell a stale frame from an idle guest.

## Clarify

- `key()` accepts a single printable character as well as the named keys, which
  is the only way to send characters when `type_text` is unsuitable.
- `click()` refuses whole-page selectors ("clicking the whole management page
  does not focus a console"); on iLO4 that means every click needs a real
  control id from the correct frame.
- iLO4 serial-over-LAN is not a BMC-only setting: BIOS console redirection must
  be enabled in RBSU. Cross-reference the `ilo4-html5-console` skill.
