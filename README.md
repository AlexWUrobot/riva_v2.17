# riva_v2.17

The first issue was that riva_init.sh spawned a child Docker container to download models, but this container used Docker's isolated bridge network instead of the host's DNS, causing wget to fail resolving api.ngc.nvidia.com. Adding --network=host to that container fixed it.
The second issue was that riva_start.sh tried to bind container ports :8888 and :50051 directly to the same host ports, but both were already occupied by mediamtx and booster-server respectively, so Docker silently refused to start the container and the script looped forever. The fix was auto-detecting conflicts at runtime and remapping to the next free host port.


主要改變與成功原因
問題根源
Riva 2.17 啟動失敗有兩個獨立的問題：

問題一：模型下載失敗（DNS 無法解析）
原因： riva_init.sh 在下載模型時，會啟動一個獨立的 Docker 容器（riva-models-download）來執行 wget，但這個容器預設使用 Docker 的內部網路，無法繼承主機的 DNS 設定，所以解析不了 api.ngc.nvidia.com。
修改： 在 riva_init.sh 中，啟動下載容器時加入 --network=host，讓它直接使用主機的網路與 DNS，就能正常連到 NGC 伺服器。

問題二：容器根本無法啟動（Port 被佔用）
原因： riva_start.sh 嘗試把容器的 :8888 和 :50051 綁到主機同號 Port，但：

:8888 已被 mediamtx 佔用
:50051 已被 booster-server 佔用

Docker 直接拒絕啟動，腳本就一直卡在「等待中」。
修改： riva_start.sh 改為自動偵測 Port 是否被占用，若被占則自動選一個空閒的 Port（也可手動指定）。

結果
容器 Port主機 Port（修改後）50051 (gRPC)5005388888890
所以客戶端連線要改用 localhost:50053，而非預設的 50051。


