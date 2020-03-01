---
title: ZooKeeper 应用场景之 Master 选举
date: 2020-02-02 10:01:00
categories: ZooKeeper
tags:
    - ZooKeeper
---
Master 选举，就是在众多机器或服务中，选举出一个最终“决定权”的领导者，来独立完成一项任务。比如有一项服务是需要对外提供服务，但是要保证高可用，我们就机会进行服务的多项部署，也就是做了一些备份，提高系统的可用性。一旦我们的主服务挂了，我们可以让其它的备份服务进行重新选举，这样我们就能使整个系统不会因服务的挂掉而造成服务不可用。

## 思路
在 ZooKeeper中，有两种方式可以实现 Master 选举：
1. 谁先创建 master 临时节点，谁就是 master，当一个 master 挂掉了，master 节点就消失了，别的节点就会监听到，就会继续去创建 master 临时节点，以此类推，利用 Zookeeper 的两个特点（一个节点只能成功创建一次、利用监听的机制）
2. 在 master 下面创建临时有序节点，那个节点最小，那个就是 master，节点挂掉，下面那个临时节点就会监听到上面的临时节点挂掉了，从而取代成为 master，以此类推，（利用 Zookeeper 创建节点临时有序的特性）

## 实现
### Curator
```java
/**
 * An example leader selector client.
 * Note that {@link LeaderSelectorListenerAdapter} which has the recommended handling for connection state issues
 * <p>
 * Master 选举参与者
 */
@Slf4j
public class LeaderSelectorParticipant extends LeaderSelectorListenerAdapter implements Closeable {
    private final String name;
    private final LeaderSelector leaderSelector;
    private final AtomicInteger leaderCount = new AtomicInteger();

    public LeaderSelectorParticipant(CuratorFramework client, String path, String name) {
        this.name = name;

        // create a leader selector using the given path for management
        // all participants in a given leader selection must use the same path
        // ExampleClient here is also a LeaderSelectorListener but this isn't required
        leaderSelector = new LeaderSelector(client, path, this);

        // for most cases you will want your instance to requeue when it relinquishes leadership
        // 保证在此实例释放领导权之后还可能获得领导权
        leaderSelector.autoRequeue();
    }

    /**
     * 尝试获取领导权
     */
    public void start() {
        // the selection for this instance doesn't start until the leader selector is started
        // leader selection is done in the background so this call to leaderSelector.start() returns immediately
        leaderSelector.start();
    }

    /**
     * 自动关闭
     */
    @Override
    public void close() {
        // 释放领导权
        leaderSelector.close();
    }

    /**
     * 当你的实例被授予领导权时调用。这个方法在你希望释放领导力之前不应该返回
     */
    @Override
    public void takeLeadership(CuratorFramework client) throws Exception {
        // we are now the leader. This method should not return until we want to relinquish leadership

        final int waitSeconds = 5 * new Random().nextInt(1) + 1;

        log.info("{} is now the leader. Waiting " + waitSeconds + " seconds...", name);
        log.info("{} has been leader {} time(s) before.", name, leaderCount.getAndIncrement());
        try {
            Thread.sleep(TimeUnit.SECONDS.toMillis(waitSeconds));
        } catch (InterruptedException e) {
            log.info("{}  was interrupted.", name);
            Thread.currentThread().interrupt();
        } finally {
            log.info("{}  relinquishing leadership.\n", name);
        }
    }
}

/**
 * Master 选举示例
 */
@Slf4j
public class LeaderSelectorExample {
    /**
     * 客户端数量
     */
    private static final int CLIENT_QTY = 10;

    /**
     * 路径
     */
    private static final String PATH = "/examples/leader";

    public static void main(String[] args) throws Exception {
        // all of the useful sample code is in ExampleClient.java

        log.info("Create {} clients, have each negotiate for leadership and then wait a random number of seconds before letting another leader election occur.", CLIENT_QTY);
        log.info("Notice that leader election is fair: all clients will become leader and will do so the same number of times.");

        List<CuratorFramework> clients = Lists.newArrayList();
        List<LeaderSelectorParticipant> examples = Lists.newArrayList();
        TestingServer server = new TestingServer();
        try {
            for (int i = 0; i < CLIENT_QTY; ++i) {
                CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
                clients.add(client);

                LeaderSelectorParticipant example = new LeaderSelectorParticipant(client, PATH, "Client #" + i);
                examples.add(example);

                client.start();
                example.start();
            }

            log.info("Press enter/return to quit\n");
            new BufferedReader(new InputStreamReader(System.in)).readLine();
        } finally {
            log.info("Shutting down...");

            for (LeaderSelectorParticipant leaderSelectorParticipant : examples) {
                CloseableUtils.closeQuietly(leaderSelectorParticipant);
            }
            for (CuratorFramework client : clients) {
                CloseableUtils.closeQuietly(client);
            }

            CloseableUtils.closeQuietly(server);
        }
    }
}
```