LSM 트리, B+ 트리, 보조 인덱스, 클러스터드 인덱스 등 다양한 개념에 대해 접하면서 느낀 점은 각자 장단점이 명확하고 트레이드 오프가 존재한다는 점이다. 특히 읽기/쓰기 성능에 대해 많이 다루고 있는 느낌이다.

---

## 프로젝트 고민

- IMap
    - Hazelcast에서 제공하는 분산 시스템 메모리 저장소
    - SeaTunnel에서는 작업 상태를 관리하는데 사용된다.

- IMap에 대한 외부 저장소를 사용
    - 분산 시스템 노드들이 모두 다운되었을 때 복구
    - SeaTunnel을 재시작할 때 기존 작업 상태를 복구

imap 에 대해서 외부 저장소를 사용할 경우,

`org.apache.seatunnel.engine.imap.storage.file.wal.writer.IFileWriter` 의 구현체들은 write() 작업 시 매번 새로운 내용을 쓰기만한다.

```java
private void write(byte[] bytes) {
      try (FSDataOutputStream out = fs.create(path, true)) {
          // Write to bytebuffer
          byte[] data = WALDataUtils.wrapperBytes(bytes);
          bf.writeBytes(data);

          // Read all bytes
          byte[] allBytes = new byte[bf.readableBytes()];
          bf.readBytes(allBytes);

          // write filesystem
          out.write(allBytes);

          // check and reset
          checkAndSetNextScheduleRotation(allBytes.length);

      } catch (Exception ex) {
          throw new IMapStorageException(ex);
      }
  }
```

- 문제점
    - 외부 저장소의 용량이 계속해서 늘어남. 1TB 까지 가는 경우도 있었음.
    - 외부 저장소를 통해 IMap을 복구할 때 시간이 매우 오래 걸림. → 재시작이 매우 오래 걸려 사용자가 불편해함.

- 고려해야할 점
    - SeaTunnel에서는 IMap에 대한 쓰기 빈도가 매우 높음.
        - 이전에 다른 기여자가 해결하려는 방향은 쓰기 시 파일을 매번 정리하는 방식이었음. → 이 방식은 쓰기 빈도가 매우 높은 SeaTunnel에서 심각한 성능 병목을 초래할 수 있는 방식이다.
        
        ```java
            public void writeInternal(IMapFileData data) throws IOException {
                List<String> dataFiles = WALDataUtils.getDataFiles(fs, parentPath, FILE_NAME);
                SequenceInputStream stream = WALDataUtils.getComposedInputStream(fs, dataFiles);
                // reset block info
                reset();
        
                // write data
                writeEntry(serializer.serialize(data));
                byte[] bytes;
                boolean encountered = false;
                while ((bytes = WALDataUtils.readNextData(stream)) != null) {
                    IMapFileData diskData = serializer.deserialize(bytes, IMapFileData.class);
        
                    if (encountered) writeEntry(serializer.serialize(diskData));
                    else if (isKeyEquals(data, diskData))
                        encountered = true; // if current data is the entry which have be updated
                    else writeEntry(serializer.serialize(diskData));
                }
                stream.close();
                commit(dataFiles);
           }
        ```
        
        - 게다가 이 방식은 컴팩션이 아니라 기존 파일을 덮어쓰는 방식이다. → 동시성 제어가 필수

이 문제를 해결하기 위해 “컴팩션” 기능을 제안(LSM 트리에서 하는 것처럼)

- 사용자는 설정값을 통해 어느 주기로 컴팩션을 진행할지 결정할 수 있다.
    - SeaTunnel 실행 중에는 imap을 통해 메모리에 올려서 사용하므로 외부 저장소에 읽기를 하는 빈도 수는 매우 적음. → 컴팩션을 위해 이전 파일에 접근해도 괜찮다.
- 또, 사용자가 client/REST API를 통해 컴팩션을 진행할 수 있도록 할 수도 있다.
