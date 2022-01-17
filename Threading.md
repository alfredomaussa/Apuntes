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
