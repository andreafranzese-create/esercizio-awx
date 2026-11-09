# AWX → Jenkins: prelievo e validazione di `nginx.conf`

## L'esercizio

Costruire una catena automatizzata tra tre strumenti:

1. **AWX** – un Job Template preleva un dato da una VM (qui `/etc/nginx/nginx.conf`).
2. **Webhook** – AWX chiama Jenkins passandogli il dato, che Jenkins salva come **artefatto**.
3. **Jenkins** – la chiamata avvia una pipeline che usa l'artefatto per una "build"
4. **Git** – il `Jenkinsfile` della pipeline viene letto da GitHub, non scritto dentro Jenkins.

## Il flusso

![Flusso dell'esercizio](img/jenkins-awx.jpg)

AWX entra nella VM via SSH e legge `nginx.conf`. Il contenuto viene codificato in **base64** (così a capo e caratteri speciali non rompono il JSON) e inviato con un POST al webhook di Jenkins. Il controller Jenkins riconosce il token, scarica il `Jenkinsfile` da GitHub e fa eseguire la pipeline sull'agent: il file viene ricostruito, archiviato come artefatto e validato con `nginx -t` dentro un container nginx.

## Il playbook (`playbook.yaml`)

| Task | Cosa fa |
|---|---|
| **Leggi il file** | `slurp` legge `nginx.conf` sulla VM e lo restituisce già in base64. A differenza di `fetch` non salva nulla sul disco del container AWX, che viene distrutto a fine job. |
| **Salva negli artifacts** | `set_stats` lega nome e contenuto al job AWX: restano visibili nella pagina del job come traccia di cosa è stato prelevato. |
| **Invia a Jenkins** | `uri` fa il POST al webhook con un JSON di tre campi. `body_format: json` costruisce il JSON e l'header. `delegate_to: localhost` fa partire la chiamata da AWX e non dalla VM. `status_code: 200` fa fallire il job se Jenkins non risponde correttamente. `awx_job_id` è iniettata da AWX in ogni job. |

## La pipeline (`Jenkinsfile`)

**`agent { label 'podman' }`** – gli stage girano sull'agent con label `podman`, non sul controller. Il controller riceve solo il webhook e conserva gli artefatti.

**`triggers`** – il plugin Generic Webhook Trigger avvia la pipeline quando arriva un POST con `token=awx-java-build` ed estrae i campi del JSON (JSONPath `$.campo`) in variabili d'ambiente.

| Stage | Cosa fa |
|---|---|
| **Verifica input** | Si ferma con un errore chiaro se mancano nome o contenuto del file(es. build lanciata a mano). |
| **Crea artefatto** | Decodifica il base64 e ricrea `src/nginx.conf` identico all'originale; `archiveArtifacts` lo copia sul controller come artefatto della build. `set +x` evita di stampare il base64 nel log. |
| **Validazione** | Avvia un container nginx con il file montato e lancia `nginx -t`: la build è verde solo se la configurazione è valida. `:Z` serve per SELinux (Rocky), che altrimenti impedisce al container di leggere il file. |

## Configurazione

### AWX
- **Installazione**: su Kubernetes (Colima) con il chart Helm `awx-operator` (`AWX.enabled=true`, service `ClusterIP`, accesso con `kubectl port-forward`).
- **Credential** `rocky-ssh` (tipo *Machine*): utente e chiave SSH della VM.
- **Inventory** `vms`: un host, l'IP della VM.
- **Project**: repo GitHub con playbook e Jenkinsfile
- **Job Template**: project + `playbook.yaml` + inventory `vms` + credential `rocky-ssh`.

### Jenkins
- **Plugin**: *Generic Webhook Trigger*.
- **Agent**: nodo con label `podman`, con Podman installato.
- **Job Pipeline**: *Pipeline script from SCM* → Git → repo GitHub → `Jenkinsfile`.

## Note

- **Perché `uri` e non una notifica AWX.** AWX ha le notifiche di tipo *Webhook*, ma in questa versione (24.6.1) l'interfaccia non permette di salvare un messaggio che legga `job.artifacts`. La stessa chiamata HTTP è stata quindi spostata nel playbook.
- **Sicurezza.** Il token è in chiaro per semplicità.