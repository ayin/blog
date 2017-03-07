Java8中提供了一个@Contended注解

# 伪共享(False Sharing)

获取Cache Line的大小：
```bash
# Mac
sysctl machdep.cpu.cache.linesize

# Linux
getconf LEVEL1_DCACHE_LINESIZE
```


[False Sharing, Cache Coherence, and the @Contended Annotation on the Java 8 VM](https://dzone.com/articles/false-sharing-cache-coherence-and-the-contended-an)
