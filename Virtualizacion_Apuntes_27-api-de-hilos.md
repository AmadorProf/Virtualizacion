# 27 · Interludio: la API de hilos (pthreads)

> Apuntes propios sobre el capítulo 27 de OSTEP (*Interlude: Thread API*). Fuente: https://ostep.org

## 27.1 Crear y esperar hilos

```c
#include <pthread.h>

void *tarea(void *arg) {
    int x = *(int *)arg;
    int *res = malloc(sizeof(int));
    *res = x * 2;
    return res;                    // valor de retorno del hilo
}

int main(void) {
    pthread_t t;
    int arg = 21;
    void *ret;
    pthread_create(&t, NULL, tarea, &arg);   // attr=NULL: por defecto
    pthread_join(t, &ret);                   // espera y recoge retorno
    // *(int*)ret == 42
}
```

- `pthread_create(&t, attr, funcion, arg)`: la función del hilo recibe un único `void*` y devuelve `void*` — el truco genérico de C para pasar/devolver cualquier cosa (empaquetar structs si hacen falta varios argumentos).
- `pthread_join(t, &ret)`: espera a que termine y recoge su retorno — el `wait()` de los hilos.
- **Error clásico nº1**: devolver (o pasar) un puntero a una variable **de la pila del hilo** — al terminar el hilo su pila desaparece y el puntero queda colgando. Retornos por heap o estructuras del llamante.
- No siempre se hace join (servidores con hilos que viven indefinidamente; hilos *detached*).

## 27.2 Locks

```c
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;   // init estática
// o dinámica: pthread_mutex_init(&m, NULL);

pthread_mutex_lock(&m);
   // sección crítica
pthread_mutex_unlock(&m);
```

- **Comprueba los códigos de retorno** (el capítulo insiste): un lock que falla en silencio = carreras misteriosas. En código de clase se envuelven en asserts (`Pthread_mutex_lock` con mayúscula, wrappers que abortan si hay error).
- Variantes: `pthread_mutex_trylock` (no bloquea: falla si está cogido) y `timedlock` (con plazo) — útiles para evitar interbloqueos (cap. 32), a usar con moderación.

## 27.3 Variables de condición

Para dormir hasta que ocurra algo (detalle completo en cap. 30):

```c
pthread_cond_t c = PTHREAD_COND_INITIALIZER;

// hilo que espera:
pthread_mutex_lock(&m);
while (!listo)                    // SIEMPRE while, nunca if
    pthread_cond_wait(&c, &m);    // suelta m al dormir; la readquiere al despertar
pthread_mutex_unlock(&m);

// hilo que avisa:
pthread_mutex_lock(&m);
listo = 1;
pthread_cond_signal(&c);
pthread_mutex_unlock(&m);
```

Las dos reglas que el capítulo subraya:

1. **wait/signal siempre con el lock cogido** — señalizar sin lock abre carreras sutiles.
2. **Recomprobar la condición en bucle (`while`)**, no `if`: el despertar no garantiza que la condición siga siendo cierta (despertares espurios, otros hilos que se adelantan).

Y el antipatrón que las motiva: sincronizar con un flag y espera activa (`while (!listo);`) — quema CPU y es propenso a errores. Nunca.

## 27.4 Compilar y consejos

`gcc -o p p.c -Wall -pthread`. El resto del capítulo es una lista de higiene: mantener la sincronización **simple**, minimizar interacciones entre hilos, inicializar siempre locks y CVs, y usar las páginas man (`man 7 pthreads`) y los sanitizers.

## Ampliación (no está en OSTEP)

- Bajo pthreads en Linux está `clone()` + **futex** (cap. 28): un mutex sin contención no entra al kernel — por eso son baratos.
- **Datos por hilo**: `__thread`/`_Thread_local` (TLS, implementado sobre el registro FS — el fósil de segmentación del cap. 16).
- C11 trajo `<threads.h>` (thrd_create…), casi sin adopción; C++ tiene `std::thread`/`std::jthread` (RAII: join automático). En la práctica, pthreads sigue siendo la lingua franca en C.
- `pthread_attr_t` permite fijar el **tamaño de pila** (por defecto ~8 MiB virtuales en Linux) — relevante con miles de hilos (y la motivación de los hilos verdes, `../virtualizacion.md` §2.4).

## Referencias

- OSTEP cap. 27.
- Butenhof, *Programming with POSIX Threads*, 1997 — el libro de referencia de pthreads.
- `man 7 pthreads`, `man 3 pthread_create`, `man 3 pthread_cond_wait`.
