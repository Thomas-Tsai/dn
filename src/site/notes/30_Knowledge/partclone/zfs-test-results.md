---
{"dg-publish":true,"permalink":"/30-knowledge/partclone/zfs-test-results/","dg-note-properties":{}}
---

# ZFS Pool Sector-level Clone 驗證記錄

## 背景與動機

Clonezilla 不原生支援 ZFS，對含 ZFS pool 的磁碟只能 fallback 到 `partclone.dd` 整碟掃描（即使 99% 空白也要全部讀取並寫入 image）。本專案目標：

**用 partclone 框架為 ZFS 提供 used-block-aware backup/restore**，只複製 pool 實際使用的 block，達到接近 `partclone.extfs` / `partclone.btrfs` 的體驗。

### 兩種可行路線（評估後選 B）

| 路線 | 工具 | 層級 | 優點 | 缺點 |
|---|---|---|---|---|
| A | `zfs send -R` | logical stream | ZFS 原生，支援增量、保留快照 | 產生的是資料流，不是 partclone image；只能 restore 到既有 ZFS pool；無法 sector-level imagize system disk |
| B | **partclone.zfs** (本專案) | sector + 自產 bitmap | 符合 clonezilla 體驗；可 image 任何 partition | 需要讀懂 ZFS 內部 metadata，工程較大 |

兩者都驗證過，本文記錄 **路線 B** 的手工驗證 → partclone 整合 → 端到端實測。

---

## 環境

| 項目 | 值 |
|---|---|
| OS | Ubuntu 24.04, kernel 6.8.0-139-generic (VM) |
| ZFS kernel module | 2.2.2-0ubuntu9.4 |
| ZFS userspace source tree | OpenZFS 2.4.99 (`/home/thomas/zfs/`，已 `./configure && make` 完成） |
| 開發用 headers/libs | `libzfslinux-dev` 2.2.2-0ubuntu9.5 (`sudo apt install libzfslinux-dev`) |
| 測試用磁碟 | `/dev/vdc` 20 GB（源）, `/dev/vdd` 20 GB（目標） |
| partclone repo | `/home/thomas/partclone/` (git master,autotools 正常） |

重要 observation：source tree 與 kernel module 版本不一致（2.4.99 vs 2.2.2）但 `libzpool`（userland 模擬）仍可工作，因為 offline 讀 metadata 不需要 ioctl。

---

## Phase 1: 手工 sector-level clone 驗證（概念證明）

目的：**先證明「用 ZFS bitmap 加上逐 range 的 `dd`」能做出 sparse-clone 且 import 后可用**，再決定要不要做 partclone 整合。

### 工具

- `/home/thomas/zfsclone.c` + `Makefile` → `zfsclone` 二進制（獨立小工具，非 partclone module)
- `/home/thomas/clone-via-bitmap.sh` → 讀 zfsclone 輸出，逐 range `dd` 到目標碟

### zfsclone 概述

```c
/* 概念 */
spa_open_rewind(pool) → traverse_pool(callback)
callback: 每個 blkptr → DVA_GET_OFFSET / DVA_GET_ASIZE → mark bitmap
補強:metaslab spacemap + log spacemap 的 ALLOC/FREE → mark/clear bitmap
輸出:text format "vdev <id> <start_byte> <len>"
```

### Phase 1 結果（成功）

```
源:vdc1  stripe pool,50 small files + 200MB big + 2 snaps + nested dataset
zfsclone 報告:used 189,114,880 bytes (~180 MB)
dd 實際複製:同上 (189,114,880 bytes)
總 20 GB → 只複製 0.88%     ←sparse 大幅節省
            ↓
zpool import testpool      ✅
zpool scrub testpool       ✅ 0B repaired, 0 errors
zfs list -t snapshot      ✅ 兩個 snapshot 都在
md5sum (全部檔案)         ✅ 與來源一致
```

關鍵細節：`clone-via-bitmap.sh` 知道 DVA offset 是 **data-area-relative**，所以在 dd 時加上 `VDEV_LABEL_START_SIZE (= 4 MiB)`:

```bash
DATA_AREA_ON_DISK=$(( PART_START + VDEV_LABEL_START_SIZE ))
dd if=$SRC of=$DST bs=4K iflag=skip_bytes skip=$(( DATA_AREA_ON_DISK + _start )) ...
```

> 這一步驗證了 ZFS「label + data」的實體布局，為下一步整合作好準備。

---

## Phase 2: 整合 partclone 框架

### 架構

```
partclone/
├── configure.ac           ← + --enable-zfs 選項
├── src/Makefile.am        ← + ENABLE_ZFS block 產生 partclone.zfs
├── src/partclone.h        ← + #define zfs_MAGIC "ZFS"
└── src/zfsclone.c         ← 新檔案,fs driver (同 extfsclone.c 架構)
    ├── fs_open()          : kernel_init + spa_open_rewind + /dev 掃描 fallback
    ├── fs_close()         : spa_close + kernel_fini
    ├── read_super_blocks():填充 fs_info (block_size, totalblock, usedblocks, device_size)
    └── read_bitmap()      :traverse_pool callback + spacemap replay → pc_set_bit()
```

`partclone.zfs` 跟其他 `partclone.<fs>` 一樣，CLI 相容，直接能用 `partclone.restore` 還原。

### Library 策略

用 **系統 `libzfslinux-dev`**（類比 extfs → `libext2fs`、ntfs → `libntfs-3g`)，**不** vendor openzfs source tree。

- configure.ac 用 `AC_CHECK_FILE` 檢查 headers(`/usr/include/libzfs/sys/spa.h` 等）+ `AC_CHECK_LIB` 連結測試（`libzpool`、`libzfs`、`libzfs_core`、`libnvpair`、`libuutil`)
- 注意：**不用** `AC_CHECK_HEADERS`，因為 OpenZFS headers 需要 `_LARGEFILE64_SOURCE`/`_GNU_SOURCE` 才能過編譯
- 連結：`-lzpool -lzfs -lzfs_core -lnvpair -luutil -lblkid -luuid -ludev -lcrypto -lzstd -lz -ldl`
- 例外：`libzpool.h` 不在 dev package，直接在程式用 `extern void kernel_init(int); extern void kernel_fini(void);` 宣告

### 編譯

```bash
cd /home/thomas/partclone
autoreconf -fi
./configure --enable-zfs --enable-extfs --enable-btrfs
# 輸出最後一行應含: zfs........... yes, unknown (uses libzfslinux-dev)
make -j
# artifact: src/partclone.zfs
```

---

## 關鍵 bug 與修正

第一次整合後端到端測試 import 出現 `FAULTED / corrupted data on vdd1 ONLINE`。逐層排查後找到：

### Bug:DVA offset 是 data-area-relative

OpenZFS 在 `module/zfs/zio.c` 的 `zio_vdev_child_io()` 有：

```c
if (vd->vdev_ops->vdev_op_leaf) {
    ASSERT0(vd->vdev_children);
    offset += VDEV_LABEL_START_SIZE;    /* = 4 MiB */
}
```

意思是：

| 概念 | 意義 | 算法 |
|---|---|---|
| **DVA offset**(`DVA_GET_OFFSET(dva)`) | data-area-relative,**不含 4 MB label area** | 物理 partition 位址 = DVA offset + 4 MB |
| **vdev label 物理位址**(zdb -l 用的） | partition-absolute | 直接用，不需 +4 MB |

初次實作把 DVA offset 當 partition-absolute 用 → bitmap 資訊整體偏移 **-4 MB** → restore 寫錯位置 → import 失敗。

### 修正

在 `partclone/src/zfsclone.c` 分兩類 helper:

```c
/* 用於 DVA / spacemap entry:data-area 進位但 ZFS IO 自動 +4 MB */
static inline void mark_range_used_dva(walk_ctx_t *ctx, uint64_t dva_off, uint64_t len) {
    mark_range_used(ctx, dva_off + VDEV_LABEL_START_SIZE, len);
}

/* 用於 vdev label / uberblock 區域:partition-absolute */
static void mark_range_used(walk_ctx_t *ctx, uint64_t off, uint64_t len) {
    /* 不加 VDEV_LABEL_START_SIZE */
}
```

所有 traverse callback / spacemap replay 改用 `mark_range_used_dva()`；標記 label 區域（開頭 4MB + 末 512KB）仍用一般 `mark_range_used()`。

### 附帶發現（ZFS 行為）

檢查 rootbp 時看到 rootbp 有 `DVA[0]`、`DVA[1]`、`DVA[2]` 三個 ditto copy，但物理碟上**只有 DVA[0] 有實際資料**,DVA[1]/DVA[2] 位置是 all-zero。這是 ZFS 的 lazy ditto write 行為：**只用 primary，僅在需要 rescue 時才寫 backup**。

所以：

- 不能只複製 DVA[1]/DVA[2],**必須覆 DVA[0]**（這點 traverse_pool 有照顧到，因為 BP_GET_NDVAS() 會給全部 3 個）
- 我們的 bitmap 原本就覆蓋全部 DVA（單看其中一個 bytes是有資料的就安全）

### Bug 2:`/dev` 掃描 fallback 會誤選 stale pool

現象：當系統同時存在兩片 ZFS-labelled 磁碟（例如 source `/dev/vdc1` 和之前 restore 過的 `/dev/vdd1`，都有可能含 "testpool" 名稱的 vdev label),fallback 掃描整個 `/dev/` 時 `zpool_find_config("testpool")` 可能挑到**錯的那個**(stale、損毀的），導致：

```
zfsclone.c: cannot open pool 'testpool' (device /dev/vdc1): No such file or directory
```

#### 修正（雙階 fallback)

```c
/* 先在「源裝置」掃 → 該裝置只有一個 candidate → 絕對不混淆 */
err = try_import_via_device(device);

/* 找不到 → 才 fallback 到整個 /dev (適合 pool 已 export 且 cachefile 不在) */
if (err == ENOENT)
    err = try_import_via_device("/dev");
```

第一階段用 `importargs_t.path = { device }` restricted scan，只會看到裝置本身；不會被其他碟干擾。

---

## 端到端測試紀錄

### Test 1：基本流程

```
[FS] zpool create testpool /dev/vdc1
[FS] zfs create testpool/fs + 寫檔 + snapshot
[PT] partclone.zfs -c /dev/vdc1 → /tmp/zfs.partclone     ←   46.9 MB used
[PT] partclone.zfs -r /tmp/zfs.partclone → /dev/vdd1     ←   restore
[FS] wipe /dev/vdc 完全 → /dev/vdd1 是唯一來源
[FS] zpool import testpool
     ✅ ONLINE
     ✅ scrub 0 errors
     ✅ zfs list snapshot: snap1, snap2 都在
     ✅ md5sum 來源 = clone
```

### Test 2：使用空間有效率

```
整碟 20 GB = 41,938,432 blocks (512B)
zfs 用 46.9 MB = 91,506 blocks (0.22%)
partclone image 檔案大小:52 MB(自動 zstd blocks_per_checksum=2048 + bitmap + 壓縮資料)
```

---

## 完整可重現範例 (reproducible example)

以下從零開始，完整走完 **建 pool → 塞資料 + md5 baseline → partclone.zfs clone → wipe source → partclone.zfs restore 到新碟 → import 驗證 md5sum** 全流程。

### 0. 前置

```bash
# 依賴
sudo apt update && sudo apt install -y libzfslinux-dev partclone

# partclone.zfs 是我們編的 (在 /home/thomas/partclone/src/)
# 依賴 zpool/zfs 命令(此 VM 用 source-tree 編的)
export ZPOOL=/home/thomas/zfs/.libs/zpool
export ZFS=/home/thomas/zfs/.libs/zfs
PC_ZFS=/home/thomas/partclone/src/partclone.zfs

# 乾淨測試碟
sudo wipefs -a /dev/vdc /dev/vdd
sudo sgdisk -Z /dev/vdc /dev/vdd
sudo sgdisk -n 1:2048:-2048 -t 1:BF01 /dev/vdc   # Linux ZFS partition type
sudo sgdisk -n 1:2048:-2048 -t 1:BF01 /dev/vdd
sudo partprobe /dev/vdc /dev/vdd
sleep 1
```

### 1. 建 pool + 塞資料 + md5 baseline

```bash
sudo $ZPOOL create -O compression=lz4 testpool /dev/vdc1
sudo $ZFS create testpool/fs

# 各式各樣資料:小檔、大檔、snapshot、刪檔後 snapshot、nested dataset
for i in 1 2 3 4 5; do
    echo "small file $i" | sudo tee /testpool/fs/s$i.txt > /dev/null
done
sudo dd if=/dev/urandom of=/testpool/fs/big01 bs=1M count=80 status=none
sudo $ZFS snapshot testpool/fs@snap1
sudo rm /testpool/fs/s1.txt /testpool/fs/s2.txt
sudo dd if=/dev/urandom of=/testpool/fs/big02 bs=1M count=30 status=none
sudo $ZFS snapshot testpool/fs@snap2
sudo $ZFS create testpool/fs/nested
sudo dd if=/dev/urandom of=/testpool/fs/nested/blob bs=1M count=20 status=none
sync
sleep 6

# 重要:md5 baseline,後面要驗證
sudo bash -c 'cd /testpool && find . -type f -exec md5sum {} \; | sort -k2' \
    > /tmp/baseline.md5
sudo $ZFS list -r testpool
sudo $ZFS list -t snapshot -r testpool
```

### 2. Export pool 並 Clone

```bash
sudo $ZPOOL export testpool
sudo rm -rf /etc/zfs   # 純合法 cache,不影響 - 讓測試走 /dev scan fallback

sudo $PC_ZFS -c -s /dev/vdc1 -o /tmp/zfs.partclone -L /tmp/clone.log
# 輸出應該是:
#   File system:  ZFS
#   Device size:   21.5 GB = 41938432 Blocks
#   Space in use:  ~160 MB = ~308,000 Blocks
#   Free Space:    ~21 GB
#   Block size:   512 Byte
```

### 3. Wipe 源碟，只剩 image 檔

```bash
sudo wipefs -a /dev/vdc /dev/vdc1
sudo dd if=/dev/zero of=/dev/vdc bs=1M status=none 2>&1 | tail -1
# vdc 現在完全無資料
```

### 4. Restore image 到 /dev/vdd1

```bash
sudo $PC_ZFS -r -s /tmp/zfs.partclone -o /dev/vdd1 -L /tmp/restore.log
```

### 5. Mount 新碟並驗證 md5sum

```bash
# 直接用 pool 名 import,vdd1 上 vdev label 記著原始 path=/dev/vdc1 但
# ZFS 用 vdev_guid 取到實際 device = /dev/vdd1
sudo $ZPOOL list 2>&1 | head -3   # 應該 no pools available (尚未 import)
sudo $ZPOOL import -d /dev/ testpool
sudo $ZPOOL status testpool
# 應該:
#   pool: testpool
#  state: ONLINE
# config:
#       NAME        STATE     READ WRITE CKSUM
#       testpool    ONLINE       0     0     0
#         vdd1      ONLINE       0     0     0

sudo $ZFS list -r testpool
sudo $ZFS list -t snapshot -r testpool
# snap1, snap2 應該都在

# Mount 並檢查 md5
sudo $ZFS mount -a
ls -la /testpool/fs/
sudo bash -c 'cd /testpool && md5sum -c /tmp/baseline.md5'
# 預期輸出:
#   ./fs/big01: OK
#   ./fs/big02: OK
#   ./fs/s3.txt: OK
#   ./fs/s4.txt: OK
#   ./fs/s5.txt: OK
#   ./fs/nested/blob: OK
#   ./fs/snap1/big01 (or 等價路徑): OK
#   ...
```

### 6. 更嚴格：scub checksum verify

```bash
sudo $ZPOOL scrub testpool
sleep 5
sudo $ZPOOL status testpool
# 應看到:
#   scan: scrub repaired 0B in MM:SS with 0 errors on Wed ... 2026
#   errors: No known data errors
```

### 7. 結束清理

```bash
sudo $ZPOOL destroy testpool
sudo wipefs -a /dev/vdd /dev/vdd1
rm -f /tmp/zfs.partclone /tmp/baseline.md5 /tmp/clone.log /tmp/restore.log
```

---

## v1 限制與後續工作

### 支援範圍

| 情況                                                        | v1 支援   | 說明                                                                                  |
| --------------------------------------------------------- | ------- | ----------------------------------------------------------------------------------- |
| Stripe pool on partition (`vdev_children == 1`, vdev 是分區） | ✅       | 最常見的 Linux-on-ZFS root disk 場景                                                      |
| Mirror pool                                               | ❌       | 需先 mirror split，或多裝置處理                                                              |
| RAIDZ                                                     | ❌       | parity striping 未支援                                                                 |
| Whole-disk pool (`zpool create ... /dev/sda`)             | ❌       | vdev label 實際在 sda 上的 GPT partition,clonezilla 用 `/dev/sda` 直接傳進來會找不到；需全碟 GPT parse |
| Multi-vdev concat                                         | ❌       | vdev id != 0                                                                        |
| Compressed / encrypted pool                               | ✅（不解壓縮） | raw sector 複製，import 後 ZFS 自己解 LZ4/ZSTD/crypto                                      |
| Pool 在 active / exported / `zpool import -N` 狀態           | ✅       | 自動用 libzpool userland 讀；fallback 到 `/dev` 掃描 + spa_import                           |

### 已知 ZFS 行為注意事項

1. **vdev label `path` 不變**:clone 後 `zpool status` 看起來還是舊路徑（例如 `/dev/sda1`),ZFS 用 vdev GUID 找到實際 device.建議 restore 完先 export 原 pool 再 import 新的（如果兩個同時插著）。
2. **ditto DVA[1]/DVA[2] 可能是 all-zero**:bitmap 必須覆蓋 DVA[0] 就好；我們有覆蓋全部，無差。
3. **讀 active pool 是 fuzzy**:zfsclone 在 mounted+active 時仍可跑，但 bitmap 是「抓那一刻的 near-consistent」，可能含有 pending free/alloc；正式 clone 建議 export first。
4. **ZFS feature flag 相容性**:2.2.2 libzpool 讀 2.3+ 的 pool，如果 pool 啟用了新 feature flag(如某些 zil_rewrite）可能 import 失敗；屆時會報 unsupported feature。

### 後續工作（roadmap)

| 項 | 優先度 | 說明 |
|---|---|---|
| 進 clonezilla-live ISO validation | high | 把 partclone.zfs 放進 rescue ISO，走 full clonezilla workflow |
| GPT whole-disk parse | medium | 讓 `partclone.zfs /dev/sda` 自動找到 ZFS partition 並正確偏移 bitmap |
| `tests/zfs.test` 接上 partclone test framework | low | 用 losetup + 建 pool 自動化 make check |
| Mirror/RAIDZ/multi-vdev | low | 需理解 vdev parent/child 映射 + 多碟對齊 clonezilla 一次一碟 model |
| Incremental bitmap(diff 兩次 snapshot) | low | 用 `zfs clone`/`bookmark` 產生 changed-block ranges |

---

## 參考檔案 / artifact

| 檔案 | 角色 |
|---|---|
| `/home/thomas/zfsclone.c` | Phase 1 獨立測試 binary（非 partclone module) |
| `/home/thomas/Makefile` | Phase 1 用獨立 gcc 連結 source-tree libzpool |
| `/home/thomas/clone-via-bitmap.sh` | Phase 1 sparse dd 腳本 |
| `/home/thomas/test-zfsclone.sh` | Phase 1 zfsclone vs `zdb -b bp allocated` 驗證腳本 |
| `/home/thomas/partclone/src/zfsclone.c` | Phase 2 partclone fs driver （主要 deliverable) |
| `/home/thomas/partclone/src/zfsclone.h` | header stub |
| `/home/thomas/partclone/configure.ac` | + `--enable-zfs` |
| `/home/thomas/partclone/src/Makefile.am` | + `ENABLE_ZFS` block |
| `/home/thomas/partclone/src/partclone.h` | + `zfs_MAGIC "ZFS"` |
| `/home/thomas/partclone/src/partclone.zfs` | build artifact（可直接用） |

---

## 附錄：`zfsclone.c` 主要實作關鍵 API

| API | 來源 | 用途 |
|---|---|---|
| `kernel_init(SPA_MODE_READ)`/`kernel_fini()` | libzpool.so （自 `extern`) | userland 初始化/清理 |
| `spa_open_rewind(name, &spa, FTAG, NULL, NULL)` | libzpool | 離線打開 pool(metadata only) |
| `zpool_find_config()` + `spa_import()` | libzutil/libzfs | pool 不在 cachefile 時，掃 /dev/ 找裝置並導入到 userspace namespace |
| `zpool_read_label(fd, &cfg, &nlabels)` | libzutil | 讀 vdev label 拿 pool name（不 import) |
| `traverse_pool(spa, 0, TRAVERSE_PRE \| TRAVERSE_PREFETCH_METADATA, cb, arg)` | libzpool | 走每個 live blkptr |
| `BP_GET_NDVAS`/`DVA_GET_OFFSET`/`DVA_GET_ASIZE` | sys/spa.h | 取每個 bp 的物理位址 |
| `BP_IS_HOLE`/`BP_IS_REDACTED`/`BP_IS_EMBEDDED` | sys/spa.h | 過濾不需佔空間的 bp |
| `space_map_length`/`dmu_read`/`SM_*_DECODE`/`SM2_*_DECODE` | libzpool | 手動解 spacemap ALLOC/FREE 補孤兒 |
| `spa_feature_is_active(spa, SPA_FEATURE_LOG_SPACEMAP)` | libzpool | 處理 v2 spacemap |
| `pc_set_bit`/`pc_clear_bit`/`pc_init_bitmap` | partclone bitmap.h | 填 partclone bitmap |

> 注意：所有 ZFS `DVA_GET_OFFSET` / `SM_OFFSET` / `SM2_OFFSET` 都是 **data-area-relative**(0 表示 data area 起點 = partition + 4 MB)，不是 partition-absolute。在 `zio_vdev_child_io()` 中 ZFS 自動 +4 MB。
