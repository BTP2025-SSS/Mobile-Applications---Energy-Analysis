1. Sleep with delay

// BETTER: Sleep instead of busy looping
while (!dataAvailable) {
    try {
        Thread.sleep(100);  // sleep for 100 ms
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}
processData();


2. Wait/Notify mechanism

class SharedResource {
    private boolean dataAvailable = false;

    public synchronized void produceData() {
        dataAvailable = true;
        notify(); // wake up waiting thread
    }

    public synchronized void consumeData() throws InterruptedException {
        while (!dataAvailable) {
            wait(); // sleep until producer notifies
        }
        // proceed safely
        System.out.println("Data consumed!");
    }
}

3. Blocking Queues

import java.util.concurrent.*;

public class Example {
    private static BlockingQueue<String> queue = new ArrayBlockingQueue<>(1);

    public static void main(String[] args) throws Exception {
        // Producer
        new Thread(() -> {
            try {
                Thread.sleep(1000); // simulate delay
                queue.put("Hello");
            } catch (Exception e) {}
        }).start();

        // Consumer (blocks instead of busy-waiting)
        String data = queue.take(); // waits efficiently
        System.out.println("Got data: " + data);
    }
}
