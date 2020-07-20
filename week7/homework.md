## 作业一：

**以下两题，至少选做一题**

- 性能压测的时候，随着并发压力的增加，系统响应时间和吞吐量如何变化，为什么？
- 用你熟悉的编程语言写一个 web 性能压测工具，输入参数：URL，请求总次数，并发数。输出参数：平均响应时间，95% 响应时间。用这个测试工具以 10 并发、100 次请求压测 www.baidu.com。

性能压测的时候，随着并发压力的增加，系统的响应时间会逐步增加，当到达系统的最大负载点时，响应时间会大幅增加，直至系统奔溃无响应。系统的吞吐量会随着并发的增加而快速上升，到达负载点后会缓慢增加，继续增加并发量，系统的吞吐量会逐渐降低，继续增加并发，系统可能奔溃。

```java
public class CapacityTesting {

    private String url;
    private int totalNum;
    private ExecutorService pool;

    public CapacityTesting(String url, int totalNum, ExecutorService executorService) {
        this.url = url;
        this.totalNum = totalNum;
        this.pool = executorService;
    }

    double[] test(){
        CountDownLatch cdl = new CountDownLatch(totalNum);
        Queue<Long> timeQueue = new ConcurrentLinkedQueue<>();
        for(int i=0;i<totalNum;i++){
            pool.execute(new Runnable() {
                @Override
                public void run() {
                    HttpURLConnection con = null;
                    long startTime = System.nanoTime();
                    // Make HTTP request to the specified URL
                    try {
                        URL link = new URL(url);
                        con = (HttpURLConnection) link.openConnection();
                        con.setRequestMethod("GET");
                        BufferedReader in  = new BufferedReader(new InputStreamReader(con.getInputStream(), "utf-8"));
                        String result = "";
                        String line;
                        while ((line = in.readLine()) != null) {
                            result += line;
                        }
                        System.out.println(result);
                    } catch (IOException e) {
                        e.printStackTrace();
                    } finally {
                        if (con != null) {
                            con.disconnect();
                        }
                    }
                    long endTime = System.nanoTime();
                    timeQueue.add(endTime - startTime);
                    System.out.println(endTime - startTime);
                    cdl.countDown();
                }
            });
        }
        try {
            cdl.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        List<Long> times = new ArrayList<>(timeQueue);
        double avgResponseTime = getAvg(times) / 1000000000.0;
        double ninetyFivePercentileTime = get95Percentile(times) / 1000000000.0;
        return new double[]{avgResponseTime, ninetyFivePercentileTime};
    }

    private static double getAvg(List<Long> times) {
        long total = 0;
        for (long time : times) {
            total += time;
        }
        return total / (float)times.size();
    }
    private static long get95Percentile(List<Long> times) {
        Collections.sort(times);
        int index = (int) Math.ceil(95 / 100.0 * times.size());
        return times.get(index-1);
    }

    public static void main(String[] args) {
        ExecutorService pool = Executors.newFixedThreadPool(10);
        CapacityTesting capacityTesting = new CapacityTesting("http://www.baidu.com",100,pool);
        double[] metrics = capacityTesting.test();
        System.out.println("The average response time is " + metrics[0] + " seconds and the 95 " +
                "percentile response time is " + metrics[1] + " seconds");
        pool.shutdown();
    }
}
```

