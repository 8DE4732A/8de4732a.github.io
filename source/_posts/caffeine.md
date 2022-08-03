---
title: caffeine
date: 2022-08-01 17:33:02
tags: cache
---

[ConcurrentLinkedHashMap设计与代码解析_飞星魂的博客-CSDN博客_concurrentlinkedhashmap](https://blog.csdn.net/u014410538/article/details/78657657)

[高性能缓存 Caffeine 原理及实战](https://zhuanlan.zhihu.com/p/348695568)

[GitHub - JCTools/JCTools](https://github.com/JCTools/JCTools)

[W-TinyLFU论文阅读与原理分析​](https://www.modb.pro/db/218970)

[Caffeine 详解 -- Caffeine 的 Window TinyLfu](https://zhuanlan.zhihu.com/p/329774936)

[https://www.jianshu.com/p/39438fc0b6a8](https://www.jianshu.com/p/39438fc0b6a8)

```java
	@NonNegative
  public int frequency(E e) {
    if (isNotInitialized()) {
      return 0;
    }

    int hash = spread(e.hashCode());
    int start = (hash & 3) << 2;
    int frequency = Integer.MAX_VALUE;
    for (int i = 0; i < 4; i++) {
      int index = indexOf(hash, i);
      int count = (int) ((table[index] >>> ((start + i) << 2)) & 0xfL);
      frequency = Math.min(frequency, count);
    }
    return frequency;
  }

public void increment(E e) {
    if (isNotInitialized()) {
      return;
    }

    int hash = spread(e.hashCode());
		// long类型64bit 可以放16个4bit
		// start 值：0、4、8、12
    int start = (hash & 3) << 2;

    // Loop unrolling improves throughput by 5m ops/s
		// 根据种子取4种不同的hash结果
    int index0 = indexOf(hash, 0);
    int index1 = indexOf(hash, 1);
    int index2 = indexOf(hash, 2);
    int index3 = indexOf(hash, 3);

		// 4种不同的深度取得bit位 +1
    boolean added = incrementAt(index0, start);
    added |= incrementAt(index1, start + 1);
    added |= incrementAt(index2, start + 2);
    added |= incrementAt(index3, start + 3);
		// 如果达到最大值 就重置 除以2
    if (added && (++size == sampleSize)) {
      reset();
    }
  }

```

```java
	/** Evicts entries if the cache exceeds the maximum. */
  @GuardedBy("evictionLock")
  void evictEntries() {
    if (!evicts()) {
      return;
    }
		//从window驱逐转移到main probation里
    int candidates = evictFromWindow();
    evictFromMain(candidates);
  }

	/**
   * Evicts entries from the window space into the main space while the window size exceeds a
   * maximum.
   *
   * @return the number of candidate entries evicted from the window space
   */
  @GuardedBy("evictionLock")
  int evictFromWindow() {
    int candidates = 0;
    Node<K, V> node = accessOrderWindowDeque().peek();
    while (windowWeightedSize() > windowMaximum()) {
      // The pending operations will adjust the size to reflect the correct weight
      if (node == null) {
        break;
      }

      Node<K, V> next = node.getNextInAccessOrder();
      if (node.getPolicyWeight() != 0) {
        node.makeMainProbation();
        accessOrderWindowDeque().remove(node);
        accessOrderProbationDeque().add(node);
        candidates++;

        setWindowWeightedSize(windowWeightedSize() - node.getPolicyWeight());
      }
      node = next;
    }

    return candidates;
  }
	

	/**
   * Determines if the candidate should be accepted into the main space, as determined by its
   * frequency relative to the victim. A small amount of randomness is used to protect against hash
   * collision attacks, where the victim's frequency is artificially raised so that no new entries
   * are admitted.
   *
   * @param candidateKey the key for the entry being proposed for long term retention
   * @param victimKey the key for the entry chosen by the eviction policy for replacement
   * @return if the candidate should be admitted and the victim ejected
   */
  @GuardedBy("evictionLock")
  boolean admit(K candidateKey, K victimKey) {
		//LFU 根据victim的使用频率决定是否保留
    int victimFreq = frequencySketch().frequency(victimKey);
    int candidateFreq = frequencySketch().frequency(candidateKey);
    if (candidateFreq > victimFreq) {
      return true;
    } else if (candidateFreq <= 5) {
      // The maximum frequency is 15 and halved to 7 after a reset to age the history. An attack
      // exploits that a hot candidate is rejected in favor of a hot victim. The threshold of a warm
      // candidate reduces the number of random acceptances to minimize the impact on the hit rate.
			// 防止HASH冲突
      return false;
    }
    int random = ThreadLocalRandom.current().nextInt();
    return ((random & 127) == 0);
  }

	/** Promote the node from probation to protected on an access. */
  @GuardedBy("evictionLock")
  void reorderProbation(Node<K, V> node) {
    if (!accessOrderProbationDeque().contains(node)) {
      // Ignore stale accesses for an entry that is no longer present
      return;
    } else if (node.getPolicyWeight() > mainProtectedMaximum()) {
      reorder(accessOrderProbationDeque(), node);
      return;
    }

    // If the protected space exceeds its maximum, the LRU items are demoted to the probation space.
    // This is deferred to the adaption phase at the end of the maintenance cycle.
    setMainProtectedWeightedSize(mainProtectedWeightedSize() + node.getPolicyWeight());
    accessOrderProbationDeque().remove(node);
    accessOrderProtectedDeque().add(node);
    node.makeMainProtected();
  }

	/** Updates the node's location in the page replacement policy. */
  @GuardedBy("evictionLock")
  void onAccess(Node<K, V> node) {
    if (evicts()) {
      K key = node.getKey();
      if (key == null) {
        return;
      }
			//增加LFU
      frequencySketch().increment(key);
      if (node.inWindow()) {
        reorder(accessOrderWindowDeque(), node);
      } else if (node.inMainProbation()) {
			//重排序 从probation -> protected
        reorderProbation(node);
      } else {
        reorder(accessOrderProtectedDeque(), node);
      }
      setHitsInSample(hitsInSample() + 1);
    } else if (expiresAfterAccess()) {
      reorder(accessOrderWindowDeque(), node);
    }
    if (expiresVariable()) {
      timerWheel().reschedule(node);
    }
  }
```