# 大文件上传与断点续传的后端设计

## 引言
大文件上传的难点不在“传”，而在“稳”。当文件体积达到数百 MB 甚至 GB 级别时，网络抖动、浏览器崩溃、服务重启都可能导致上传失败。要提升可用性，后端必须支持分片上传、断点续传、幂等校验和最终合并。在 [架构设计](https://go-ayx-app.com.cn) 上，应将上传协议、状态管理与文件存储解耦，避免单点状态丢失。

## 核心原理分析
断点续传的核心是“可恢复状态”。通常流程是：客户端先计算文件唯一标识（如 `md5/sha256`），再按固定大小切片上传，每个分片携带 `uploadId`、`chunkIndex`、`chunkHash`。后端保存分片到临时目录或对象存储，并记录已完成分片列表。这样即使中途失败，也能只补传缺失部分。

设计上要关注四点：
1. **幂等性**：同一分片重复上传不能造成脏数据。
2. **状态持久化**：分片进度应落库或写入 Redis，避免服务重启后丢失。
3. **并发控制**：允许多分片并发上传，但合并阶段必须串行。
4. **校验与清理**：合并前校验分片完整性，成功后删除临时文件。

## 代码示例
下面是一个简化的 Go 示例：接收分片并在全部完成后合并，适合落地为后端核心逻辑。

```go
package main

import (
	"fmt"
	"io"
	"os"
	"path/filepath"
)

func saveChunk(uploadID string, index int, r io.Reader) error {
	dir := filepath.Join("tmp", uploadID)
	if err := os.MkdirAll(dir, 0755); err != nil {
		return err
	}
	f, err := os.Create(filepath.Join(dir, fmt.Sprintf("%d.part", index)))
	if err != nil {
		return err
	}
	defer f.Close()
	_, err = io.Copy(f, r)
	return err
}

func mergeChunks(uploadID, target string, total int) error {
	out, err := os.Create(target)
	if err != nil {
		return err
	}
	defer out.Close()

	for i := 0; i < total; i++ {
		partPath := filepath.Join("tmp", uploadID, fmt.Sprintf("%d.part", i))
		part, err := os.Open(partPath)
		if err != nil {
			return err
		}
		if _, err := io.Copy(out, part); err != nil {
			part.Close()
			return err
		}
		part.Close()
	}
	return os.RemoveAll(filepath.Join("tmp", uploadID))
}
```

## 总结
大文件上传的本质，是把一次高风险的大请求拆成可恢复、可校验、可重试的小请求。后端只要把分片存储、状态管理和合并流程设计清楚，就能显著提升稳定性与用户体验。对于生产环境，建议进一步接入对象存储、Redis 进度表和异步合并任务，以支撑更高并发和更大文件规模。

## 相关技术资源
- https://go-ayx-app.com.cn
