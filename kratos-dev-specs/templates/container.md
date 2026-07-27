# 容器配置模板

容器内配置目录固定为 `/data/conf`。外部配置可来自不同主机路径、ConfigMap、Secret 或 PVC，但镜像内的 `-conf` 和 `mountPath` 必须一致。

```dockerfile
# 构建产物路径和 ENTRYPOINT 必须从当前 Dockerfile 事实发现。
COPY --from=builder /workspace/bin/server ./server
VOLUME ["/data/conf"]
CMD ["./server", "-conf", "/data/conf"]
```

不要凭模板假定 Docker 构建产物位置或镜像是否已有 `ENTRYPOINT`。若镜像有 `ENTRYPOINT ["./server"]`，Dockerfile 通常只保留 `CMD ["-conf", "/data/conf"]`；若没有 ENTRYPOINT，则上例的完整 `CMD` 是一条可执行命令。以实际 Dockerfile 为准，不能同时写出冲突的二进制路径。

```yaml
containers:
  - name: <project>
    # image 没有 ENTRYPOINT 时：
    command: ["./server"]
    args: ["-conf", "/data/conf"]
    volumeMounts:
      - name: config
        mountPath: /data/conf
        readOnly: true
volumes:
  - name: config
    configMap:
      name: <project>-config
```

Kubernetes `command` 覆盖镜像 ENTRYPOINT，`args` 覆盖 CMD。因此：镜像已经以 `ENTRYPOINT ["./server"]` 指定二进制时，Deployment/StatefulSet 只写 `args: ["-conf", "/data/conf"]`；镜像没有 ENTRYPOINT 时，才同时写上述 `command` 和 `args`。绝不能让两者拼成 `./server ./server -conf /data/conf`。

变更时逐项检查 Dockerfile、Helm `values` 与 templates、Deployment/StatefulSet、ConfigMap/Secret 的挂载和 CI 的镜像运行命令。确认每个容器内 `mountPath` 与 `-conf` 均为 `/data/conf`，同时确认外部来源差异不会被错误强制统一。
