# Tracepoint 使用方法

本文描述使用 perf-tools 的 tpoint 抓取内核 tracepoint 事件的步骤。

## 使用步骤

1. 下载并解压 perf-tools：
   ```bash
   wget 'https://github.com/brendangregg/perf-tools/archive/refs/tags/v1.0.tar.gz' && tar -zxvf perf-tools-1.0.tar.gz
   ```
2. 抓取指定 tracepoint：
   ```bash
   ./perf-tools-1.0/bin/tpoint -H {tracepoint}
   ```