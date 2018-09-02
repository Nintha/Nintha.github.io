---
title: Ticktocker:简易倒计时工具类
date: 2017-07-16
---

code:
```
public class TickTocker {
    private final static ConcurrentLinkedQueue<Ele> queue = new ConcurrentLinkedQueue<>();
    private Consumer<Long> callback = null;
    public final int period = 1000 * 5; //10s
    public final long expirTime = 1000 * 12; //1min

    public static TickTocker init(Consumer<Long> callback) {
        TickTocker tickTocker = new TickTocker();
        tickTocker.start(callback);
        return tickTocker;
    }

    public void add(long key) {
        queue.offer(new Ele(key));
    }

    private void start(Consumer<Long> callback) {
        this.callback = callback;
        Timer timer = new Timer();
        timer.scheduleAtFixedRate(new TimerTask() {
            public void run() {
                doJob();
            }
        }, 1000, period);
    }

    private void doJob() {
        //获取队列元素
        Ele e = queue.peek();
        //空队列什么都不执行
        if (e == null) {
            System.out.println(LocalDateTime.now() + " - Queue is empty.");
            return;
        }
        //判断是否超时
        if (isExpired(e.getCtime(), expirTime)) {
            System.out.println(LocalDateTime.now() + " - KEY:" + e.getKey() + " is expired.");
            queue.poll(); //从队列中移出第一个元素
            callback.accept(e.getKey()); //执行回调函数
            doJob();
        } else {
//            System.out.println(LocalDateTime.now() + " - Do nothing.");
        }

    }

    private boolean isExpired(long ctime, long expirTime) {
//        System.out.println("ctime: " + ctime + ", expirTime: " + expirTime + ", now: " + System.currentTimeMillis());
        return ctime + expirTime < System.currentTimeMillis();
    }

    class Ele {
        private long ctime;
        private long key;

        public Ele(long key) {
            this.key = key;
            this.ctime = System.currentTimeMillis();
        }

        public long getKey() {
            return key;
        }

        public void setKey(long key) {
            this.key = key;
        }

        public void setCtime(long ctime) {
            this.ctime = ctime;
        }

        public long getCtime() {
            return ctime;
        }
    }

    public static void main(String[] args){
        Random random = new Random();
        System.out.println(LocalDateTime.now() + " - Program started.");
        TickTocker tickTocker = TickTocker.init(key -> {
            System.out.println("Do something for " + key);
        });
        ExecutorService exec = Executors.newCachedThreadPool();
        for (int i = 100; i < 110; i++){
            final int key = i;
            exec.execute(()->{
                try {
                    Thread.sleep(random.nextInt(20000));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(LocalDateTime.now() + " - add KEY: " + key);
                tickTocker.add(key + 0L);
            });
        }
        exec.shutdown();
    }
}

```