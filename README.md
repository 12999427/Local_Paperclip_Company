Installazione
Prerequisiti
Node.js 18+ (verifica con node --version)
npm o npx disponibile nel terminale
Installazione e primo avvio
# Esegui l'onboarding (installa e configura tutto in automatico)
npx paperclipai onboard --yes
Il flag --yes accetta tutti i default: modalità local_trusted, accessibile solo da localhost, nessuna autenticazione richiesta.

Al termine, avvia il server:

npx paperclipai run
Il server gira su http://localhost:3100 — la UI è accessibile da browser, ma per questa guida lo controlliamo da Claude Code.

