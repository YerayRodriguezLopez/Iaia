# Iaia

# 1. Guia tècnica d'instal·lació i configuració d'OVOS Bookworm (català)

## Índex
1. Versió d’OVOS i imatge base
2. Política de backup (abans de qualsevol instal·lació)
3. Wake word i openWakeWord
4. STT: fasterwhisper
5. TTS: piper i veu catalana
6. Dialeg i normalització de text
7. Feedback sonor i limitacions
8. Persona i LLM remot
9. Llenguatge del sistema i pipeline d’intents
10. PHAL i skills
11. Annexos i enllaços

---

## 1. Versió d’OVOS i imatge base

Imatge base utilitzada: **[raspOVOS‑Bookworm arm64 offline – 2025‑06‑18](https://github.com/OpenVoiceOS/raspOVOS/releases/tag/raspOVOS-bookworm-arm64-offline-2025-06-18)**.  
Eina de gravació: **[Raspberry Pi Imager 1.8.4 estable](https://downloads.raspberrypi.com/imager/ )** (versió arxivada).

---

## 2. Política de backup (obligatòria)

Abans de qualsevol instal·lació o canvi de configuració cal crear una **còpia de seguretat completa** del sistema i 
validar el procediment de restauració. Aquesta regla s’aplica a **totes les seccions** d’instal·lació (STT, TTS, wake word, persona, skills, etc.).

### Punts de backup obligats

- Backup de la **imatge sencera** del sistema.
- Backup dels **fitxers de configuració d’OVOS** (`mycroft.conf` o equivalent).
- Backup dels **plugins i models** instal·lats localment.
- Backup dels **directorios de veus TTS** i registre d’arrencada.
- Recuperar‑se a aquest punt de referència cada cop que un canvi de `pip` o plugin trenqui el flux.

---

## 3. Wake word i openWakeWord

El sistema utilitza la **paraula d’activació (wake word)** `yaya`, detectada amb el plugin **`ovos‑ww‑plugin‑openwakeword`**, que no ve instal·lat per defecte. Aquest plugin requereix un model local `.tflite` entrenat explícitament.

### Entrenament del model `yaya`

- El model `yaya` es genera a partir del **notebook Colab de `openWakeWord`** (per exemple, el notebook de 
`training_models.ipynb`).
- El Colab és pensat principalment per a anglès, però gràcies a la brevetat del wake word (`yaya`) funciona bé en català.
- Temps d’entrenament i generació del model: entre **una i dues hores**, pot ser més segons la mostra i la configuració 
de la rutina de Colab.

### Instal·lació de `openwakeword` i `ovos‑ww‑plugin‑openwakeword`

```bash
# Actualitzar ovos i instal·lar el plugin de wake word openwakeword
ovos-update
ovos-install ovos-ww-plugin-openwakeword
``` 

O bé com alternativa amb pip install:

```bash
pip install ovos-ww-plugin-openwakeword
``` 

### Estructura de fitxers del model

El model generat s’ha de copiar al camí que tens definit, per exemple:

- `/home/ovos/.local/share/mycroft/openwakeword/models/yaya.tflite`

Aquesta és la ruta que s’ha d’indicar a la configuració de `hotwords`.

### Configuració de listener i hotwords

```json
"listener": {
  "wake_word": "yaya",
  "confirm_listening": true,
  "confirm_utterance": true,
  "record_utt_pre": 0.5,
  "record_utt_post": 0.8,
  "timeout": 5,
  "sec_per_buff": 0.1,
  "bits": 16,
  "multiplier": 1.0,
  "max_queued_utterances": 1
},
"hotwords": {
  "yaya": {
    "module": "ovos-ww-plugin-openwakeword",
    "models": [
      "/home/ovos/.local/share/mycroft/openwakeword/models/yaya.tflite"
    ],
    "inference_framework": "tflite",
    "threshold": 0.55,
    "listen": true,
    "active": true
  }
}
```

### Descart de Vosk per a wake word

- `Vosk` es va utilitzar inicialment com a **STT**, i per la detecció de wake word.
- El comportament de detecció amb Vosk era **inconsistent** i no adequat per a aquest rol.
- El sistema definitiu utilitza **`openwakeword`** per a detecció de wake word, que és més estable i controlable 
per a latència i llindar de detecció.

---

## 4. STT: fasterwhisper i català

El sistema utilitza el plugin **`ovos‑stt‑plugin‑fasterwhisper`** per a reconeixement de veu en català. 
Aquest plugin és multi‑idioma i, amb el model `small`, ofereix una qualitat adequada sense dependre de servidor remot.

### Comandaments de referència

El sistema OVOS ja disposa de fasterwhisper, però en cas que no, es pot instal·lar amb la següent comanda:

```bash
ovos-install ovos-stt-plugin-fasterwhisper
```

O bé com alternativa amb pip install:

```bash
pip install ovos-stt-plugin-fasterwhisper
```

### Configuració STT

```json
"stt": {
  "module": "ovos-stt-plugin-fasterwhisper",
  "fallback_module": "",
  "ovos-stt-plugin-fasterwhisper": {
    "model": "small",
    "use_cuda": false,
    "num_workers": 4,
    "cpu_threads": 4,
    "compute_type": "int8"
  }
}
```

Aquesta configuració és la base de referència per a STT en català. Qualsevol prova nova hauria d’aplicar‑se sobre un 
backup previ creat abans d’instal·lar el plugin o canviar el model.

---

## 5. TTS: piper i veu `ca_ES-upc_ona-medium`

El sistema utilitza el plugin **`ovos‑tts‑plugin‑piper`** amb la veu catalana `ca_ES‑upc_ona‑medium`, 
que forma part del catàleg oficial de Piper.

### Comandaments de referència per a la veu catalana

```bash
# Instal·lar el plugin Piper
ovos-install ovos-tts-plugin-piper
```

O bé com alternativa amb pip install:

```bash
# Instal·lar el plugin Piper
pip install ovos-tts-plugin-piper
```

```bash
# Crear el directori de la veu
mkdir -p ~/.local/share/piper/voices/ca_ES-upc_ona-medium/

# Descarregar el model .onnx
wget -O ~/.local/share/piper/voices/ca_ES-upc_ona-medium/voice.onnx \
  https://huggingface.co/rhasspy/piper-voices/resolve/main/ca/ca_ES/upc_ona/medium/ca_ES-upc_ona-medium.onnx?download=true

# Descarregar el fitxer de configuració
wget -O ~/.local/share/piper/voices/ca_ES-upc_ona-medium/config.json \
  https://huggingface.co/rhasspy/piper-voices/resolve/main/ca/ca_ES/upc_ona/medium/ca_ES-upc_ona-medium.onnx.json?download=true
```

### Configuració TTS

```json
"tts": {
  "module": "ovos-tts-plugin-piper",
  "fallback_module": "",
  "ovos-tts-plugin-piper": {
    "voice": "ca_ES-upc_ona-medium",
    "path": "~/.local/share/piper/voices/ca_ES-upc_ona-medium/",
    "use_cuda": false,
    "length_scale": 1.0,
    "noise_scale": 0.667,
    "noise_w": 0.8,
    "piper_leading_silence": 0.5
  }
}
```

Aquesta és la configuració de referència per a la síntesi de veu en català.

---

## 6. Dialeg i normalització de text

El sistema utilitza el plugin **`ovos‑dialog‑normalizer‑plugin`** per a transformar el text abans que arribi a TTS. 
Aquest plugin només cal habilitar‑lo; no requereix configuració addicional.

### Configuració de normalització de diàleg

```json
"dialog_transformers": {
  "ovos-dialog-normalizer-plugin": {}
}
```

Aquesta línia habilita el plugin i permet substituir símbols per a unes paraules més naturals en la síntesi de veu.

#### Modificació de la skill `ovos‑skill‑weather`

La skill de meteorologia original enviava símbols com `,` i `.` a la TTS, cosa que produïa una pronunciació poc neta. 
Per a resoldre‑ho, el codi de la skill ha estat modificat per convertir:

- `,` → `"coma"`
- `.` → `"punt"`

Aquest canvi assegura que la temperatura i les mesures es pronuncien clarament, sense que el TTS intenti “dir” el símbol 
directament.

---

## 7. Feedback sonor i limitacions

El sistema de so ha tingut problemes amb alguns plugins, que no poden gestionar correctament l’entrada i sortida de 
sons simultànies. Actualment, **l’únic feedback de so que funciona de manera estable** és el de detecció de wake word, configurat als fitxers definits a `sounds`.

### Configuració de sounds

```json
"sounds": {
  "start_listening": "/home/ovos/snd/on-sfx.wav",
  "end_listening": "/home/ovos/snd/off-sfx.wav",
  "finish_listening": "/home/ovos/snd/off-sfx.wav",
  "acknowledgement": "/home/ovos/snd/off-sfx.wav"
}
```

Qualsevol intent de definir àudio addicional (per exemple, per a confirmacions de skills) hauria de ser provat 
exhaustivament per evitar interferències amb el micròfon i la síntesi de veu.

---

## 8. Persona i LLM remot

El component de persona `iaia` connecta amb un **LLM remot allotjat per un altre equip** de la vostra organització. 
Aquesta integració requereix configuració explícita i un **system prompt** que s’afegeix a les peticions.

### Comportament de la persona i el LLM

- El sistema intenta resoldre la consulta amb skills locals.
- Si no hi ha resolució, deriva la petició al **fallback de persona** cap al LLM remot.
- El sistema aplica un **system prompt** (no visible per l’usuari final) a cada petició, per mantenir el context 
empresarial i de seguretat.

### Exemple de configuració de persona

```json
"persona": {
  "handle_fallback": true,
  "default_persona": "iaia",
  "ovos-persona-http-client": {
    "endpoint": "<URL_O_HOST_REMOT>",
    "model": "qwen-30b-moe",
    "fallback": true
  }
}
```

El valor de `endpoint` és privat i ha de coincidir amb el servei exposat per l’equip de LLM.

### Ubicació de la configuració de Persona

```
/home/ovos/.config/ovos_persona/iaia.json
```

---

## 9. Llenguatge del sistema i pipeline d’intents

El sistema està configurat per treballar en català (`ca-es`) a tot el pipeline d’intents, sense referència explícita a 
cap “versió de configuració” particular; el comportament es deriva directament del fitxer de configuració compartit.

### Configuració de intents i pipeline

```json
"intents": {
  "ovos-m2v-pipeline": {
    "model": "Jarbas/ovos-model2vec-intents-LaBSE",
    "conf_high": 0.7,
    "conf_medium": 0.5,
    "conf_low": 0.15,
    "ignore_intents": []
  },
  "persona": {
    "handle_fallback": true,
    "default_persona": "iaia"
  },
  "pipeline": [
    "ocp_high",
    "stop_high",
    "converse",
    "padatious_high",
    "ovos-m2v-pipeline-high",
    "adapt_high",
    "stop_medium",
    "adapt_medium",
    "common_qa",
    "fallback_medium",
    "ovos-persona-pipeline-plugin-low",
    "fallback_low"
  ],
  "padatious": {
    "stem": false,
    "cast_to_ascii": true,
    "domain_engine": false
  }
}
```

Aquest pipeline gestiona intents, comunica amb el LLM i retorna la resposta via TTS en català. Cada canvi d’aquesta zona 
cal fer‑se sobre un backup prèvi del sistema.

---

## 10. PHAL i skills

El sistema deshabilita alguns components de maquinari no necessaris i aplica una política de **blacklist de skills** 
per mantenir‑lo estable i senzill.

### Configuració PHAL i skills

```json
"PHAL": {
  "ovos-PHAL-plugin-camera": {
    "enabled": false
  }
},
"skills": {
  "blacklisted_skills": [
    "ovos-skill-parrot.openvoiceos",
    "ovos-skill-speedtest.openvoiceos",
    "skill-ovos-somafm.openvoiceos",
    "ovos-skill-hello-world.openvoiceos",
    "ovos-skill-days-in-history.openvoiceos",
    "ovos-skill-ip.openvoiceos",
    "ovos-skill-date-time.openvoiceos",
    "skill-ovos-spelling.openvoiceos",
    "ovos-skill-moviemaster.openvoiceos",
    "ovos-skill-laugh.openvoiceos",
    "ovos-skill-wolfie.openvoiceos",
    "ovos-skill-wordnet.openvoiceos",
    "ovos-skill-confucius-quotes.openvoiceos",
    "ovos-skill-ddg.openvoiceos",
    "ovos-skill-local-media.openvoiceos",
    "skill-ovos-randomness.openvoiceos",
    "ovos-skill-youtube-music.openvoiceos",
    "ovos-skill-dictation.openvoiceos",
    "ovos-skill-naptime.openvoiceos",
    "ovos-skill-wikihow.openvoiceos",
    "ovos-skill-pyradios.openvoiceos",
    "ovos-skill-news.openvoiceos",
    "ovos-skill-camera.openvoiceos",
    "ovos-skill-word-of-the-day.openvoiceos",
    "ovos-skill-icanhazdadjokes.openvoiceos",
    "ovos-skill-iss-location.openvoiceos",
    "ovos-skill-personal.openvoiceos",
    "ovos-skill-number-facts.openvoiceos"
  ]
}
```

Només les skills bàsiques de meteorologia i alertes són efectives; la resta de resolucions es fa a través de la 
**persona i el LLM**.

## 11. Modificació de la skill "weather": parse de símbols.

Arxiu modificat:
```
/home/ovos/.venvs/ovos/lib/python3.11/site-packages/ovos_skill_weather/__init__.py
```

La modificació realitzada optimitza el text per a la pronunciació correcta del TTS Piper, convertint els símbols de 
puntuació bàsics a la paraula completa. Això evita que Piper pronunciï "325" en comptes de "32 coma 5".

---

## 12. Annexos i enllaços

Tots els fitxers de configuració i recursos es documenten a continuació, quan el llicència ho permet.

### Fitxers i enllaços de referència

- **Imatge raspOVOS Bookworm 2025‑06‑18**  
  https://github.com/OpenVoiceOS/raspOVOS/releases/tag/raspOVOS-bookworm-arm64-offline-2025-06-18

- **Raspberry Pi Imager 1.8.4 estable**  
  https://downloads.raspberrypi.com/imager/

- **Piper voices – català `ca_ES‑upc_ona‑medium`**  
  https://huggingface.co/rhasspy/piper-voices/tree/main/ca/ca_ES/upc_ona/medium

- **Projecte `openWakeWord` (Colab i models)**  
  https://github.com/dscripka/openWakeWord

- **Plugin `ovos‑tts‑plugin‑piper`**  
  https://github.com/OpenVoiceOS/ovos-tts-plugin-piper

- **Plugin `ovos‑stt‑plugin‑fasterwhisper`**  
  https://github.com/OpenVoiceOS/ovos-stt-plugin-fasterwhisper

- **Documentació tècnica d’OVOS (pipelines i plugins)**  
  https://openvoiceos.github.io/ovos-technical-manual/
