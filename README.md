# RepoARSW

Thread → representa un hilo de ejecución.

Runnable → interfaz funcional que define la tarea de un hilo (run).

Callable<V> → similar a Runnable pero devuelve un valor y puede lanzar excepciones.

Future<V> → resultado pendiente de un Callable.

📌 Ejemplo simple con Thread y Runnable:

class MiTarea implements Runnable {
    @Override
    public void run() {
        System.out.println("Soy un hilo corriendo: " + Thread.currentThread().getName());
    }
}

public class EjemploThread {
    public static void main(String[] args) {
        Thread t1 = new Thread(new MiTarea());
        t1.start();  // arranca el hilo
    }
}




--------



synchronized (palabra clave) → exclusión mutua básica.

Semaphore → contador de permisos; controla cuántos hilos acceden a un recurso.

Lock / ReentrantLock → control explícito de bloqueo y desbloqueo.

Condition → permite que hilos esperen señales (await/signal).

CountDownLatch → espera a que otros hilos terminen cierta cantidad de tareas.

CyclicBarrier → hace que varios hilos esperen entre sí hasta que todos lleguen a un punto.

📌 Ejemplo con Semaphore:


import java.util.concurrent.Semaphore;

class Recurso {
    private Semaphore sem = new Semaphore(1); // solo 1 hilo a la vez

    public void usar() {
        try {
            sem.acquire(); // pedir permiso
            System.out.println(Thread.currentThread().getName() + " usando recurso");
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            sem.release(); // liberar
        }
    }
}

--------
ExecutorService → pool de hilos para ejecutar tareas.

Executors → fábrica de pools (newFixedThreadPool, newCachedThreadPool, etc.).

ScheduledExecutorService → permite programar tareas periódicas.

CompletableFuture → programación asíncrona y composición de tareas.
import java.util.concurrent.*;

public class EjemploExecutor {
    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(2);

        Callable<String> tarea = () -> {
            Thread.sleep(1000);
            return "Resultado de " + Thread.currentThread().getName();
        };

        Future<String> futuro = executor.submit(tarea);
        System.out.println("Esperando resultado...");
        System.out.println(futuro.get()); // bloquea hasta que termine

        executor.shutdown();
    }
}



---------

import java.math.BigInteger;
import java.util.concurrent.*;
import java.util.*;

/**
 * BBPParallelPi
 * 
 * Calcula dígitos hexadecimales (base 16) de pi usando la fórmula BBP y
 * paraleliza el cálculo por rangos con hilos.
 *
 * Uso:
 *   java BBPParallelPi <numDigits> <numThreads>
 *
 * Ejemplo:
 *   java BBPParallelPi 50 4
 *
 * Resultado: imprime 50 dígitos hexadecimales de pi (a partir de la posición 0).
 */
public class BBPParallelPi {

    // Calcula el dígito hexadecimal en la posición n (0-indexed) usando BBP
    // Devuelve el valor 0..15 (un nibble)
    public static int piHexDigit(long n) {
        // Suma las cuatro series de la fórmula BBP
        double x = 4.0 * series(1, n) - 2.0 * series(4, n) - series(5, n) - series(6, n);
        x = x - Math.floor(x);
        int hexDigit = (int) Math.floor(16.0 * x);
        return hexDigit & 0xF;
    }

    // Calcula la suma parcial de la serie S_j = sum_{k=0..inf} 16^{n-k} / (8k + j)
    // Implementación que computa la parte finita (k=0..n) con modular exponentiation
    // y la parte tail (k=n+1..inf) con potencias de 16^{-t} usando double.
    private static double series(int j, long n) {
        // Parte finita: k = 0..n
        double s = 0.0;
        for (long k = 0; k <= n; k++) {
            long denom = 8 * k + j;
            // compute 16^{n-k} mod denom
            long power = n - k;
            long t = modPow16(power, denom); // devuelve 16^{power} mod denom
            s += (double) t / (double) denom;
            // Solo mantenemos fracción de s
            s = s - Math.floor(s);
        }

        // Parte infinita: k = n+1..inf
        double t = 0.0;
        double pow16 = Math.pow(16.0, (double) (n + 1));
        // Terms: 16^{n-k} = 16^{-(k-n)} for k>n, equivalently 1 / 16^{k-n}
        // We'll sum en forma directa con potencias negativas
        for (long k = n + 1; ; k++) {
            long denom = 8 * k + j;
            double term = Math.pow(16.0, (double) (n - k)) / (double) denom; // 16^{n-k}/denom
            if (term < 1e-17) { // criterio de parada (suficiente para ~50-100 digits)
                break;
            }
            t += term;
        }

        s += t;
        s = s - Math.floor(s);
        return s;
    }

    // Calcula (16^exp) mod m de forma eficiente. Si exp<0 no se usará (en nuestro uso exp>=0).
    // Devuelve un long 0..m-1. Usa BigInteger para seguridad en m grande.
    private static long modPow16(long exp, long m) {
        if (m == 1) return 0;
        // Usa exponenciación modular: 16^exp mod m
        BigInteger base = BigInteger.valueOf(16);
        BigInteger modulus = BigInteger.valueOf(m);
        BigInteger result = base.modPow(BigInteger.valueOf(exp), modulus);
        return result.longValue();
    }

    // Worker que calcula un rango de dígitos hex y los guarda en un arreglo compartido
    static class DigitWorker implements Callable<Void> {
        private final long startIdx;
        private final long endIdx; // exclusive
        private final int[] output; // compartido, ya dimensionado
        private final int offset; // offset en el arreglo "output" para escritura

        DigitWorker(long startIdx, long endIdx, int[] output, int offset) {
            this.startIdx = startIdx;
            this.endIdx = endIdx;
            this.output = output;
            this.offset = offset;
        }

        @Override
        public Void call() {
            int pos = offset;
            for (long i = startIdx; i < endIdx; i++, pos++) {
                int d = piHexDigit(i);
                output[pos] = d;
            }
            return null;
        }
    }

    // Convierte arreglo de nibbles (0..15) a cadena hex
    private static String nibblesToHex(int[] data) {
        StringBuilder sb = new StringBuilder();
        for (int v : data) {
            sb.append(Integer.toHexString(v).toUpperCase());
        }
        return sb.toString();
    }

    public static void main(String[] args) throws Exception {
        if (args.length < 2) {
            System.out.println("Uso: java BBPParallelPi <numDigits> <numThreads>");
            System.out.println("Ejemplo: java BBPParallelPi 50 4");
            return;
        }

        int numDigits = Integer.parseInt(args[0]);
        int numThreads = Integer.parseInt(args[1]);
        if (numDigits <= 0 || numThreads <= 0) {
            System.out.println("numDigits y numThreads deben ser > 0");
            return;
        }

        // Array compartido para resultados
        int[] result = new int[numDigits];

        ExecutorService exec = Executors.newFixedThreadPool(numThreads);
        List<Future<Void>> futures = new ArrayList<>();

        // Distribuir rangos lo más parejo posible
        int base = numDigits / numThreads;
        int rem = numDigits % numThreads;
        int written = 0;
        for (int t = 0; t < numThreads; t++) {
            int chunk = base + (t < rem ? 1 : 0);
            if (chunk == 0) break;
            long start = written;
            long end = written + chunk;
            DigitWorker w = new DigitWorker(start, end, result, written);
            futures.add(exec.submit(w));
            written += chunk;
        }

        // Esperar terminación
        for (Future<Void> f : futures) f.get();
        exec.shutdown();

        String hex = nibblesToHex(result);
        System.out.println("Dígitos hex (n=" + numDigits + "):");
        System.out.println(hex);

        // Si quieres, imprimir espacios cada 8 nibble (4 bytes) para legibilidad
        StringBuilder grouped = new StringBuilder();
        for (int i = 0; i < hex.length(); i++) {
            grouped.append(hex.charAt(i));
            if ((i+1) % 8 == 0 && (i+1) < hex.length()) grouped.append(' ');
        }
        System.out.println("Formato agrupado:");
        System.out.println(grouped.toString());
    }
}



