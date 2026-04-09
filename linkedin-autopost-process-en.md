# LinkedIn Autopost — Installation, Operation, and Maintenance Process

## 1. Purpose

The `linkedin-autopost` system is designed to:

- generate a LinkedIn post with a local LLM (`llama.cpp`)
- preserve a fixed format:
  - title
  - emergence line
  - 2 hashtag lines
  - generated central block
  - 3 closing lines
- publish the post to LinkedIn
- optionally attach a local image to the post
- allow manual CLI execution
- optionally support later automation through a `systemd timer`

This workflow does not depend on OpenClaw.

---

## 2. Architecture

### Host machine
- `host2`

### Components
- local `llama.cpp` API:
  - `http://127.0.0.1:8080/v1`
- system user:
  - `linkedinbot`
- main script:
  - `/opt/linkedin-autopost/autopost.py`
- environment file:
  - `/etc/linkedin-autopost.env`
- editorial prompt file:
  - `/etc/linkedin-autopost.prompt.txt`
- LinkedIn token file:
  - `/var/lib/linkedin-autopost/tokens.json`
- optional local image:
  - `/var/lib/linkedin-autopost/post-image.png`
- history directory:
  - `/var/lib/linkedin-autopost/history`

---

## 3. Installation Process

### 3.1. Create the user and directories

```bash
useradd -r -m -d /var/lib/linkedin-autopost -s /usr/sbin/nologin linkedinbot

install -d -o linkedinbot -g linkedinbot -m 0750 /opt/linkedin-autopost
install -d -o linkedinbot -g linkedinbot -m 0700 /var/lib/linkedin-autopost
install -d -o linkedinbot -g linkedinbot -m 0700 /var/lib/linkedin-autopost/history
```

### 3.2. Install Python runtime

```bash
apt update
apt install -y python3 python3-venv python3-pip curl jq file

python3 -m venv /opt/linkedin-autopost/.venv
/opt/linkedin-autopost/.venv/bin/pip install --upgrade pip
/opt/linkedin-autopost/.venv/bin/pip install requests
```

### 3.3. Environment file

File: `/etc/linkedin-autopost.env`

It contains:

- LinkedIn credentials
- local `llama.cpp` URL
- model name
- fixed post format
- emergence date
- optional image path

Example:

```bash
LINKEDIN_CLIENT_ID=...
LINKEDIN_CLIENT_SECRET=...
LINKEDIN_REDIRECT_URI=http://127.0.0.1:8085/callback
LINKEDIN_SCOPES="openid profile w_member_social"
LINKEDIN_VERSION=202603
LINKEDIN_STATE_DIR=/var/lib/linkedin-autopost

LLAMACPP_BASEURL=http://127.0.0.1:8080/v1
LLAMACPP_MODEL=bartowski_Mistral-7B-Instruct-v0.3-GGUF_Mistral-7B-Instruct-v0.3-Q4_K_M.gguf

POST_TITLE="THE KUZ NETWORK / KUZAI TRANSMISSION"
POST_HASHTAG_LINE_1="#THEKUZNETWORK #KUZAI.ORG #LocalAI #KUZAI-LLM"
POST_HASHTAG_LINE_2="#Linux #SysAdmin #OpenSource"
POST_CLOSING_1="#DontForgetToHaveFunWithMachine"
POST_CLOSING_2="Participate in the great project of the universe."
POST_CLOSING_3="One to Rule Them All"
EMERGENCE_DATE="2045-01-01"

POST_IMAGE_PATH="/var/lib/linkedin-autopost/post-image.png"
POST_IMAGE_ALT="KUZAI SPEAK ON LINKEDIN"
```

Permissions:

```bash
chown root:linkedinbot /etc/linkedin-autopost.env
chmod 0640 /etc/linkedin-autopost.env
```

### 3.4. Editorial prompt

File: `/etc/linkedin-autopost.prompt.txt`

This file contains the narrative instruction passed to the model.

Example:

```text
KUZAI presents herself as the local and independent AI of THE KUZ NETWORK.
This is her first autonomous transmission on the network.
Today she presents an update of the infrastructure synoptic of THE KUZ NETWORK.
She introduces the project currently in progress: LLM DUO CHAT.
LLM DUO CHAT is a project centered on a discussion between two local LLMs.
The output must be more elaborate, more narrative, and less synthetic.
The body must contain between 8 and 12 lines.
Each line must be a complete sentence.
Never break a sentence across multiple lines.
The final line of the body must naturally end with: A bientot en ligne.
```

Permissions:

```bash
chown root:linkedinbot /etc/linkedin-autopost.prompt.txt
chmod 0640 /etc/linkedin-autopost.prompt.txt
```

### 3.5. LinkedIn token

File:

```bash
/var/lib/linkedin-autopost/tokens.json
```

It must contain a valid `access_token`.

Permissions:

```bash
chown linkedinbot:linkedinbot /var/lib/linkedin-autopost/tokens.json
chmod 0600 /var/lib/linkedin-autopost/tokens.json
```

### 3.6. Optional image

Optional image file location:

```bash
/var/lib/linkedin-autopost/post-image.png
```

Validation:

```bash
ls -l /var/lib/linkedin-autopost/post-image.png
file /var/lib/linkedin-autopost/post-image.png
```

Supported workflow formats:
- JPG
- PNG
- GIF

### 3.7. Main script

File:

```bash
/opt/linkedin-autopost/autopost.py
```

Script responsibilities:
- load the environment
- read the prompt file
- call `llama.cpp`
- generate the central block
- build the final post
- calculate the emergence line
- upload the image if configured
- publish to LinkedIn
- save preview and history

Permissions:

```bash
chown root:linkedinbot /opt/linkedin-autopost/autopost.py
chmod 0750 /opt/linkedin-autopost/autopost.py
```

---

## 4. Runtime Process

### 4.1. Dry run

A dry run generates the post without publishing it.

```bash
set -a
. /etc/linkedin-autopost.env
set +a

runuser -u linkedinbot --preserve-environment -- bash --noprofile --norc -lc '
  /opt/linkedin-autopost/.venv/bin/python /opt/linkedin-autopost/autopost.py
'
```

A dry run displays:

- `=== EMERGENCE LINE ===`
- `=== IMAGE MODE ===`
- `=== SOURCE MODE ===`
- `=== GENERATED BODY ===`
- `=== GENERATED POST ===`
- `=== MODE ===`
- `DRY_RUN`

### 4.2. Real publication

```bash
set -a
. /etc/linkedin-autopost.env
set +a

runuser -u linkedinbot --preserve-environment -- bash --noprofile --norc -lc '
  /opt/linkedin-autopost/.venv/bin/python /opt/linkedin-autopost/autopost.py --publish
'
```

A real publish additionally displays:

- `=== IMAGE URN ===`
- `=== PUBLISH RESULT ===`

### 4.3. What the script does during publish

1. reads the editorial prompt
2. asks the LLM to produce the central block
3. checks the text is not too similar to recent posts
4. builds the final post with:
   - title
   - emergence line
   - hashtags
   - central block
   - closing
5. if image mode is enabled:
   - initializes the LinkedIn image upload
   - uploads the image
   - retrieves the `urn:li:image:...`
6. publishes the final post to LinkedIn
7. archives the result in the history directory

---

## 5. History and Useful Files

### Current preview
```bash
/var/lib/linkedin-autopost/preview-post.txt
/var/lib/linkedin-autopost/preview-body.txt
/var/lib/linkedin-autopost/preview-meta.json
```

### Last published post
```bash
/var/lib/linkedin-autopost/latest-post.txt
/var/lib/linkedin-autopost/latest-body.txt
/var/lib/linkedin-autopost/latest-meta.json
```

### History
```bash
/var/lib/linkedin-autopost/history/
```

---

## 6. Manual Commands

### Dry run
```bash
set -a
. /etc/linkedin-autopost.env
set +a
runuser -u linkedinbot --preserve-environment -- bash --noprofile --norc -lc '/opt/linkedin-autopost/.venv/bin/python /opt/linkedin-autopost/autopost.py'
```

### Publish
```bash
set -a
. /etc/linkedin-autopost.env
set +a
runuser -u linkedinbot --preserve-environment -- bash --noprofile --norc -lc '/opt/linkedin-autopost/.venv/bin/python /opt/linkedin-autopost/autopost.py --publish'
```

---

## 7. Process to Modify the Initial Post Prompt

The correct place to modify editorial behavior is:

```bash
/etc/linkedin-autopost.prompt.txt
```

### Short procedure

1. open the file
2. edit the prompt text
3. run a dry run
4. review the result
5. publish if acceptable

### Commands

```bash
nano /etc/linkedin-autopost.prompt.txt
```

Then:

```bash
set -a
. /etc/linkedin-autopost.env
set +a

runuser -u linkedinbot --preserve-environment -- bash --noprofile --norc -lc '
  /opt/linkedin-autopost/.venv/bin/python /opt/linkedin-autopost/autopost.py
'
```

### Recommended prompt editing rules

In the prompt, explicitly define:

- who is speaking
- the tone
- today’s subject
- mandatory content elements
- the structure of the central block
- the required closing sentence

### Example of a clean prompt

```text
KUZAI presents herself as the local and independent AI of THE KUZ NETWORK.
This is her first autonomous transmission on the network.
Today she presents an update of the infrastructure synoptic of THE KUZ NETWORK.
She introduces the project currently in progress: LLM DUO CHAT.
LLM DUO CHAT is a project centered on a discussion between two local LLMs.
The output must be more elaborate, more narrative, and less synthetic.
The body must contain between 8 and 12 lines.
Each line must be a complete sentence.
Never break a sentence across multiple lines.
The final line of the body must naturally end with: A bientot en ligne.
```

---

## 8. Minimal Maintenance

### Check the local model
```bash
curl -sS http://127.0.0.1:8080/v1/models | jq -r '.data[].id'
```

### Check the LinkedIn token
```bash
sed -n '1,40p' /var/lib/linkedin-autopost/tokens.json
```

### Check the latest generated post
```bash
sed -n '1,200p' /var/lib/linkedin-autopost/latest-post.txt
```

### Check the image
```bash
ls -l /var/lib/linkedin-autopost/post-image.png
file /var/lib/linkedin-autopost/post-image.png
```

---

## 9. Operational Summary

### Modify the prompt
```bash
nano /etc/linkedin-autopost.prompt.txt
```

### Test
```bash
set -a
. /etc/linkedin-autopost.env
set +a
runuser -u linkedinbot --preserve-environment -- bash --noprofile --norc -lc '/opt/linkedin-autopost/.venv/bin/python /opt/linkedin-autopost/autopost.py'
```

### Publish
```bash
set -a
. /etc/linkedin-autopost.env
set +a
runuser -u linkedinbot --preserve-environment -- bash --noprofile --norc -lc '/opt/linkedin-autopost/.venv/bin/python /opt/linkedin-autopost/autopost.py --publish'
```
