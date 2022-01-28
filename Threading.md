Una guía muy completa: [Intro to python threading](https://realpython.com/intro-to-python-threading/), alternativa es [asyncio](https://docs.python.org/3/library/asyncio.html)
* [event-wait-method-with-example](https://www.includehelp.com/python/event-wait-method-with-example.aspx)
* [python multithreading](https://www.tutorialspoint.com/python/python_multithreading.htm)
* [stackoverflow: Is there any way to kill a Thread?](https://stackoverflow.com/q/323972/13636459)
* [stackoverflow: How to stop daemon thread?](https://stackoverflow.com/q/41131117/13636459)
* [Different ways to kill a thread](https://www.geeksforgeeks.org/python-different-ways-to-kill-a-thread/)

Respuesta:
Alternativa utilizando `threading`:
``` python
import threading
import time

def thread_process(id):
    print(f"Thread_Process {id}: comenzando")
    time.sleep(10)
    print(f"Thread {id}: terminando")

def thread_animation(id):
    clock=0
    while True:
        clock+=1
        time.sleep(1)
        print(f"thread_animation {id}: {clock}")

if __name__ == "__main__":
    print("Main    : Antes de crear el thread de procesos")
    x = threading.Thread(target=thread_process, args=(1,))
    y = threading.Thread(target=thread_animation, args=(2,),daemon=True)
    print("Main    : Antes de lanzar el thread de procesos")
    x.start()
    y.start()
    print("Main    : Esperar a que hilo termine")
    x.join()
    print("Main    : Los procesos han terminado")
```

Código adaptado de: [Intro to python threading](https://realpython.com/intro-to-python-threading/)

la función `thread_process` tiene un `time.sleep(10)` que simula la duración del proceso demorado, mientras un hilo está ejecutando esas tareas hay otro hilo ejecutando la animación `thread_animation` para que finalice el `while true` al finalizar el programa se activa el parámetro `daemon`. 

Para comprobar el esperar a que se termine de ejecutar `thread_process` usamos `x.join()`

La salida de consola es:
``` python
Main    : Antes de crear el thread de procesos
Main    : Antes de lanzar el thread de procesos
Thread_Process 1: comenzando
Main    : Esperar a que hilo termine
thread_animation 2: 1
thread_animation 2: 2
thread_animation 2: 3
thread_animation 2: 4
thread_animation 2: 5
thread_animation 2: 6
thread_animation 2: 7
thread_animation 2: 8
thread_animation 2: 9
Thread 1: terminando
Main    : Los procesos han terminado
```

[Documentación oficial](https://docs.python.org/3/library/threading.html)

Acá hay un ejemplo usando banderas: [https://www.bogotobogo.com/python/Multithread/python_multithreading_Synchronization_Condition_Objects_Producer_Consumer.php](https://www.bogotobogo.com/python/Multithread/python_multithreading_Synchronization_Condition_Objects_Producer_Consumer.php)
``` python 
import threading
import time
import logging

logging.basicConfig(level=logging.DEBUG,
                    format='(%(threadName)-9s) %(message)s',)

def consumer(cv):
    logging.debug('Consumer thread started ...')
    with cv:
    	logging.debug('Consumer waiting ...')
        cv.wait()
        logging.debug('Consumer consumed the resource')

def producer(cv):
    logging.debug('Producer thread started ...')
    with cv:
        logging.debug('Making resource available')
        logging.debug('Notifying to all consumers')
        cv.notifyAll()

if __name__ == '__main__':
    condition = threading.Condition()
    cs1 = threading.Thread(name='consumer1', target=consumer, args=(condition,))
    cs2 = threading.Thread(name='consumer2', target=consumer, args=(condition,))
    pd = threading.Thread(name='producer', target=producer, args=(condition,))

    cs1.start()
    time.sleep(2)
    cs2.start()
    time.sleep(2)
    pd.start()
```

# Parte 2
``` python
# %% Import modules
import os # Module: Path handling
import glob # Module: Searching files
import datetime # Module: date formatting
import json # Module: load configs
import threading # Allow running different codes in parallel (check files, wait 'q', process files)
import time # Formatting date
from queue import Queue

from bs4 import BeautifulSoup # Module: scraping xml
import xlwings as xw  # Module: read and write excel with python
from openpyxl import load_workbook # Module: read and write excel with python
import watchdog.events # Module, enable moving files detector
import watchdog.observers # Module, enable moving files detector
# lxml

currentRow=0
stack=Queue()
cv=threading.Condition()

with open('config.json','r') as f:
    configInput=json.load(f)
# %% Input parameters
folderInput=configInput["folderInput"] or 'input' # Local path for input files, where *.xml are located
folderOutput= configInput["folderOutput"] or  'processed' # Local path for output/processed files, where *.xml will be located after processed
excelFileName=configInput["excelFileName"] or  "extraction_example.xlsx" # Excel to fill

# Create folders if they don't exist
if not os.path.exists(folderInput):
    os.mkdir(folderInput)
if not os.path.exists(folderOutput):
    os.mkdir(folderOutput)

# %% Event handler
class Handler(watchdog.events.PatternMatchingEventHandler):
    def __init__(self):
        # Set the patterns for PatternMatchingEventHandler
        watchdog.events.PatternMatchingEventHandler.__init__(self, patterns=['*.xml'],
                                                             ignore_directories=True, case_sensitive=False)
  
    def on_modified(self, event):
        global tempTimer
        # currentFile=event.src_path
        # print(f"Modified file: {currentFile}")
        tempTimer.cancel()
        tempTimer= threading.Timer(1,searchFiles)
        tempTimer.start()
# %% Threads

def thread_exit():
    print(f"thread_exit: begins")
    inp=''
    try:
        while  inp != 'q':
            inp=input("Enter 'q' to exit \n")
    except:
        return
def thread_process_files(id):
    global currentRow
    print(f"thread_process_files {id}: begins")
    try:
        while True:
            print(f"Updated files, checking...")
            cell_=stack.get()
            processFile(cell_)
            stack.task_done()
    except:
        print(f"Error(thread_process_files), thread_process_files {id}: finished")

def thread_event_listener(id,src_path):
    print(f"thread_event_listener {id}: begins")
    event_handler = Handler()
    observer = watchdog.observers.Observer()
    observer.schedule(event_handler, path=src_path, recursive=True)
    observer.start()
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        observer.stop()
    observer.join()
    print(f"thread_event_listener {id}: finished")
# %% Main functions

def searchFiles():
    global stack
    global files
    files=glob.glob(os.path.join(folderInput,"*.xml")) # Searching for *.xml files
    for f in files:
        stack.put(f)

def getData(currentFile):
    with open(currentFile,"r") as p:
        soup = BeautifulSoup(p.read(),'xml') # Getting structured data from xml
        # Summary data and setting which column should be placed
        data={
        "clientName":["B",soup.select_one("Dbtr Nm").text],
        "dileDate":["A",soup.select_one("CreDtTm").text],
        "paymentAmount":["C",soup.select_one("TxDtls Amt").text],
        "paymentReferenceAmount":["D",soup.select_one("Strd Ref").text],
        }
    return data

def processFile(currentFile):
    print(f"Processing: {currentFile}")
    try:
        data= getData(currentFile)
        # Saving the modified excel cells, with the same file name
        saveDataInSheet(data)
        
    except:
        print(f"Error(processFile) could not process the file {currentFile}")
    else:
        os.replace(currentFile,os.path.join(folderOutput,currentFile.split('\\')[1]))

flagVisible=None
def saveDataInSheet(data):
    global flagVisible
    global tempTimerSaveRealTime
    try:
        if excelFileName in [i.name for i in xw.books]:
            print("Ya estaba abierta")
            wb=xw.Book(excelFileName) # Open the excel workbook
            flagVisible=True
        print(xw.books)
        sheet = wb.sheets[0] # Select the excel worksheet
        currentRow=wb.sheets[0].range('A' + str(wb.sheets[0].cells.last_cell.row)).end('up').row # Set the last row number
        currentRow+=1
        # Writting the data in the temporal worksheet
        sheet.range(f"{data['clientName'][0]}{currentRow}").value=(data["clientName"][1])
        sheet.range(f"{data['dileDate'][0]}{currentRow}").value=(datetime.datetime.strptime(data["dileDate"][1], "%Y-%m-%dT%H:%M:%S"))
        sheet.range(f"{data['dileDate'][0]}{currentRow}").number_format = 'DD/MM/YY'
        sheet.range(f"{data['paymentAmount'][0]}{currentRow}").value=(float(data["paymentAmount"][1]))
        sheet.range(f"{data['paymentAmount'][0]}{currentRow}").number_format =".00"
        sheet.range(f"{data['paymentReferenceAmount'][0]}{currentRow}").value=(float(data["paymentReferenceAmount"][1]))
        
        tempTimerSaveRealTime.cancel()
        tempTimerSaveRealTime= threading.Timer(1,saveRealTime)
        tempTimerSaveRealTime.start()
    except:
        # Si no estaba abierta, abrir en secreto
        print("abrir en secreto")
        wb=load_workbook(filename=excelFileName) # Open the excel workbook
        sheet = wb.active # Select the excel worksheet
        currentRow=sheet.max_row # Set the last row number
        currentRow+=1
        # Writting the data in the temporal worksheet
        sheet[f"{data['clientName'][0]}{currentRow}"]=data["clientName"][1]
        sheet[f"{data['dileDate'][0]}{currentRow}"]=datetime.datetime.strptime(data["dileDate"][1], "%Y-%m-%dT%H:%M:%S")
        sheet[f"{data['dileDate'][0]}{currentRow}"].number_format = 'DD/MM/YY'
        sheet[f"{data['paymentAmount'][0]}{currentRow}"]=float(data["paymentAmount"][1])
        sheet[f"{data['paymentAmount'][0]}{currentRow}"].number_format =".00"
        sheet[f"{data['paymentReferenceAmount'][0]}{currentRow}"]=float(data["paymentReferenceAmount"][1])
        wb.save(excelFileName)
    return wb

def saveRealTime():
    wb=xw.Book(excelFileName)
    wb.save(excelFileName)
# %% Main
if __name__ == "__main__":
    tempTimer= threading.Timer(0,searchFiles)
    tempTimer.start()
    tempTimerSaveRealTime= threading.Timer(1,saveRealTime)

    print("Main    : Initializing threads")
    x = threading.Thread(target=thread_process_files, args=(1,),daemon=True)
    y = threading.Thread(target=thread_event_listener, args=(2,folderInput,),daemon=True)
    z = threading.Thread(target=thread_exit,daemon=True)
    print("Main    : Beginning the threads")
    x.start()
    y.start()
    z.start()
    print("Main    : Running...")
    z.join()
    print("Main    : Finishing...")

    # if not flagVisible:
        # wb=xw.Book(excelFileName) # Open the excel workbook
        # wb.close()
        # excel_app.quit()
    print("Done.")
```

### watchdog [A Simple Example](https://pythonhosted.org/watchdog/quickstart.html)
``` python
import sys
import time
import logging
from watchdog.observers import Observer
from watchdog.events import LoggingEventHandler

if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO,
                        format='%(asctime)s - %(message)s',
                        datefmt='%Y-%m-%d %H:%M:%S')
    path = sys.argv[1] if len(sys.argv) > 1 else '.'
    event_handler = LoggingEventHandler()
    observer = Observer()
    observer.schedule(event_handler, path, recursive=True)    # evento trigger: event_handler
    observer.start()
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        observer.stop()
    observer.join()
```

Revisar los eventos [acá](https://pythonhosted.org/watchdog/api.html#module-watchdog.events)

[en esta página](https://www.geeksforgeeks.org/create-a-watchdog-in-python-to-look-for-filesystem-changes/) muestran como hacerlo.
``` python
import watchdog.events
import watchdog.observers
import time
  
  
class Handler(watchdog.events.PatternMatchingEventHandler):
    def __init__(self):
        # Set the patterns for PatternMatchingEventHandler
        watchdog.events.PatternMatchingEventHandler.__init__(self, patterns=['*.csv'],
                                                             ignore_directories=True, case_sensitive=False)
  
    def on_created(self, event):
        print("Watchdog received created event - % s." % event.src_path)
        # Event is created, you can process it now
  
    def on_modified(self, event):
        print("Watchdog received modified event - % s." % event.src_path)
        # Event is modified, you can process it now
  
  
if __name__ == "__main__":
    src_path = r"C:\Users\GeeksforGeeks\PycharmProjects\requests hotel"
    event_handler = Handler()
    observer = watchdog.observers.Observer()
    observer.schedule(event_handler, path=src_path, recursive=True)
    observer.start()
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        observer.stop()
    observer.join()
```
##### Queue [Docs](https://docs.python.org/es/3/library/queue.html)
Es casi igual que usar una lista (vi que es almenos 2-10 veces más rápida):
``` python
a=[]

a.append(item) # add element
a.pop(0) # return FIFO element
```
Aunque tiene unos métodos adicionales similares a los de "threading", destaco `Queue.task_done()` y `Queue.join()` | `Queue.put(item, block=True, timeout=None)` y`Queue.get(block=True, timeout=None)`

* Respecto a threading, recuerda que `Ctrl+c` solo acaba con el thread Main, los demás siguen vivos... si esperas `ctrl+c` dentro de un tread no Main, se bloqueará sin dejar salir de consola: `in _wait_for_tstate_lock elif lock.acquire(block, timeout): KeyboardInterrupt`

##### Timer
No olvides el "start", además ...
``` python
def hello():
    print("hello, world")

t = Timer(30.0, hello)
t.start()  # after 30 seconds, "hello, world" will be printed
```
Me pareción útil:
``` python 
tempTimerSaveRealTime.cancel()
tempTimerSaveRealTime= threading.Timer(1,saveRealTime)
tempTimerSaveRealTime.start()
```

[Threading and Multiprocessing](https://docs.xlwings.org/en/stable/threading_and_multiprocessing.html)
