# FLV 播放解碼中斷 (PipelineStatus::PIPELINE_ERROR_DECODE) 原因分析與解決方案

本技術報告詳細分析了部分影音串流在使用 `flv.js` 播放時，遭遇中斷、黑畫面、並於主控台（Console）拋出 `PIPELINE_ERROR_DECODE` 或 `D3D11Status::6` 解碼錯誤的根本原因，並說明如何在 `flv.js` 核心中實作「SPS/PPS 動態組態修正與同步機制」來徹底解決此相容性問題。

---

## 1. 問題現象 (Symptom)
* 在瀏覽器中載入 FLV 影片後，影片已成功且快速完成全段緩衝（在 `SourceBuffer` 中顯示 `buffered: 0.000 - 619.833`）。
* 但在影片開始播放沒多久（大約在第 4 秒處，解碼器開始嘗試渲染影格時），播放突然中止，並於 Console 中拋出以下錯誤：
  - 視訊軌錯誤：`[player] video error: MediaError {code: 3, message: 'PipelineStatus::PIPELINE_ERROR_DECODE: video decode error!'}`
  - 底層硬體錯誤：`D3D11Status::6` (即 Windows Direct3D 11 視訊解碼加速器返回 `VDA Error 0` 崩潰)。

---

## 2. 根本原因分析 (Root Cause)

### 2.1 初始化組態與影格資料流之 SPS Level 不一致
H.264 / AVC 視訊串流在瀏覽器的 MSE (Media Source Extensions) 中播放時，其解碼參數設定（如寬高、解碼層級 Level 等）有兩個主要來源：
1. **初始化 Segment (`moov -> avcC` 箱體)**：
   來自 FLV 檔案開頭（通常是第一個 Video Tag，其 `packetType === 0`）中所封裝的 `AVCDecoderConfigurationRecord`。解碼器在載入時會**以此資訊進行初始化**。
   - 在本案有問題的影片中，此處 SPS 宣告的 Level 為 **5.1** (`avc1.4d4033`，對應 hex `0x33`)。
2. **視訊影格資料流 (`mdat` 箱體)**：
   來自後續視訊 Tag 裡（`packetType === 1`）每個影格的前置 NALU。
   - 在本案影片的第一個 Video Tag 以及後續 Tag 中，實際影格攜帶的 SPS 宣告的 Level 卻是 **3.1** (`avc1.4d401f`，對應 hex `0x1f`)。

### 2.2 解碼器安全機制防護崩潰
當瀏覽器（特別是 Chrome 與 Windows 底層硬體解碼器）使用初始化宣告的 **Level 5.1** 規格啟動後，解碼器會預期接收對應的高規格數據。然而，隨後在解碼實際影格時，資料流中傳入的卻是 **Level 3.1** 的 SPS 參數。

這種**初始化配置 (avcC) 與實際影格資料 (mdat) Level 不匹配**的狀況，會直接觸發 Chrome 底層 D3D11 硬體解碼器的安全保護或規格不對稱，導致解碼器拒絕工作並拋出 `VDA Error` 崩潰，反映在前端即為 `video decode error`。

---

## 3. 解決方案：SPS/PPS 動態組態修正與同步機制

我們在 `flv.js` 的 Demuxer ([flv-demuxer.js](file:///c:/SourceRepository/PublicGitHub/flv.js/src/demux/flv-demuxer.js)) 中實作了一套**「SPS/PPS 動態配置同步機制」**：

1. **偵測與擷取真實參數**：
   在 `flv-demuxer.js` 的 `_parseAVCVideoData` 函數中，遍歷 NALU 時，若偵測到 `SPS (Type 7)` 與 `PPS (Type 8)`：
   - 暫存其 Payload。
   - 若為首次遇到（`!this._hasUpdatedSPS`），則啟動動態更新。

2. **重新組裝正確的 `avcC` 組態**：
   使用視訊軌中擷取到的正確 SPS (Level 3.1) 和 PPS，按照 ISO 14496-15 標準，手動在記憶體中重新組裝出一個正確的 `AVCDecoderConfigurationRecord` 位元組區塊：
   ```javascript
   let avcc = new Uint8Array(11 + sps.length + pps.length);
   avcc[0] = 1;                  // configurationVersion
   avcc[1] = sps[1];             // profile
   avcc[2] = sps[2];             // profileCompatibility
   avcc[3] = sps[3];             // level (精確修正為 3.1 / 0x1f)
   avcc[4] = 0xFC | (lengthSize - 1);
   avcc[5] = 0xE1;               // numOfSPS = 1
   // ... 寫入長度與 Payload ...
   ```

3. **通知 Remuxer 與 MSE 重新初始化**：
   更新 `this._videoMetadata` 裡的 `avcc`、寬高與 `codecString` (`avc1.4d401f`)，並重新觸發並發送：
   ```javascript
   this._onTrackMetadata('video', this._videoMetadata);
   ```
   這會引導 Remuxer 產生並向 MSE 注入最新的、帶有 Level 3.1 正確 SPS 參數的 `Initialization Segment`，使解碼器以正確的規格順利完成初始化。

4. **資料流過濾與長度扣除**：
   為了維持 fMP4 資料流的純淨，視訊資料 Tag 中重複的 SPS (7) 和 PPS (8) NALU 會被設為 `discard = true` 進行濾除。
   - 濾除時，**其長度（SPS/PPS 大小 + 4 bytes 長度 prefix）將被精確地從 sample 的 `length` 與 `track.length` 中扣除**，使 `mdat` 和 `trun` box 的大小完美對齊，避免任何 Byte Offset 錯亂。

---

## 4. 驗證結論
* **驗證網頁**：`player.html` 載入影片 `http://localhost/20260613081854.flv`。
* **驗證結果**：在 Console 日誌中，當 Demuxer 偵測到真實 SPS 時，成功印出：
  `[FLVDemuxer] Updating AVCC config with correct stream SPS & PPS...`
  `[FLVDemuxer] AVCC config updated. Codec: avc1.4d401f Level: 3.1`
  隨後影片完成全段緩衝，**並能順暢無礙地播放到底，原有的解碼器崩潰（Decode Error）已彻底修復**。
