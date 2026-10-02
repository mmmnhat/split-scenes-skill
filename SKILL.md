---
name: split-scenes
description: >-
  Skill bóc tách cảnh video chính xác từng frame (-vframes), quét siêu tốc (>3.200 fps),
  soi Dense Vision Sheets, gộp tình huống trọn vẹn (SSIM), hiểu ngữ cảnh âm thanh,
  khử trùng lặp Intro / Liên video và tích hợp Premiere Pro MCP.
  Trigger: "split-scenes", "tách cảnh", "cắt cảnh", "split video", "scene detect", "premiere pro".
---

# 🎬 Split Scenes — Precision AI Scene Detection & Splitting

Skill bóc tách cảnh video chuẩn xác từng frame: quét proxy siêu tốc (>3.200 fps), phân tích Dense Vision Sheets, gộp tình huống trọn vẹn (SSIM & Palette), hiểu ngữ cảnh âm thanh & bắt đỉnh va chạm, khử trùng lặp Intro & Liên video, cắt sạch đuôi bằng `-vframes` và hỗ trợ dựng Premiere Pro MCP.

---

## 🛑 Nguyên tắc Tương tác Bắt buộc: Không tự tiện mặc định ở bất kỳ khâu nào!

> [!IMPORTANT]
> **Agent KHÔNG ĐƯỢC tự ý gán cứng đường dẫn lưu hay cách đặt tên file ở bất kỳ giai đoạn nào!**
>
> 1. **Trước khi TẢI video (Phase 1):** Bắt buộc hỏi User:
>    * 📁 **Thư mục lưu video gốc:** Tải về lưu vào đâu? (Gợi ý đường dẫn, nhưng để User quyết định).
>    * 🏷️ **Tên thư mục & Tên file:** Đặt tên là gì? (Tên gọn theo chủ đề của User hay giữ nguyên Title/ID dài ngoằng của YouTube).
>
> 2. **Ngay đầu PHASE 2 (Trước khi lập `scene_db.json`):** Bắt buộc hỏi User về quy chuẩn xuất clip:
>    * 📁 **Thư mục lưu clips:** Sau khi split sẽ lưu vào đâu? (VD: thư mục con `scenes/`, ổ cứng khác, Desktop...).
>    * 🏷️ **Tiền tố tên file (Prefix):** Bắt đầu bằng gì? (VD: `fail_`, `clip_`, `<channel>_<id>_`, hoặc tên tùy chỉnh theo dự án).
>    * 🔢 **Quy cách đánh số:** Đánh số kiểu gì? (2 chữ số `01`, 3 chữ số `001`, 4 chữ số `0001`, bắt đầu từ số mấy).
>    * 📝 **Hậu tố ngữ nghĩa (Suffix):** User muốn tên file vật lý là `prefix_số.mp4` (VD: `fail_001.mp4` — chuẩn khuyến nghị để tránh lỗi đường dẫn và ký tự tiếng Việt) hay có kèm Title tiếng Việt vào tên file (VD: `fail_001_xe_tai_dam_cao_toc.mp4`)?
>
> 3. **Tự động Require & Tự Cài Đặt Môi Trường (Zero Manual Setup):**
>    * Toàn bộ các script trong skill đều tích hợp module tự khởi động [`scripts/env_bootstrap.py`](file:///Users/mmmnhat/.gemini/antigravity/skills/split-scenes/scripts/env_bootstrap.py).
>    * Khi chạy bất kỳ tác vụ nào, hệ thống **tự động kiểm tra 100% các package cần thiết**: `opencv-python`, `scikit-image`, `numpy`, `scipy`, `librosa`, `faster-whisper`, `yt-dlp` và các công cụ CLI (`ffmpeg`, `ffprobe`).
>    * Nếu máy tính thiếu bất kỳ thư viện hay công cụ nào, script **TỰ ĐỘNG CÀI ĐẶT NGAY LẬP TỨC** (`pip install` / `brew install`) trong nền, **tuyệt đối không bắt User phải gõ lệnh cài đặt thủ công hay can thiệp terminal!**

---

## Cấu trúc thư mục & Ánh xạ `scene_db.json`

```
<User_Chosen_Download_Dir>/                        ← Thư mục video gốc do USER CHỌN ở Phase 1:
├── <User_Chosen_Name>.mp4                         ← 1. Video gốc (1080p)
├── proxy_480p.mp4                                 ← 2. Proxy H.264 ultrafast (dùng để detect)
└── <User_Chosen_Name>.info.json                   ← Metadata từ yt-dlp

<User_Chosen_Output_Dir>/                          ← Thư mục lưu clips do USER CHỌN ở đầu Phase 2:
├── fail_001.mp4                                   ← Tên file vật lý ngắn gọn, chuẩn OS
├── fail_002.mp4
└── ...
```

### Chuẩn cấu trúc một bản ghi trong `scene_db.json`:
Dù tên file ngoài ổ đĩa chỉ là `prefix_số` (`fail_001.mp4`), database vẫn lưu trữ đầy đủ `title`, `action`, `category`, `tags` để Premiere Pro MCP hiển thị:
```json
{
  "scene_id": 1,
  "file_name": "fail_001.mp4",
  "file_path": "/Volumes/External/clips/fail_001.mp4",
  "start_f": 3348,
  "end_f": 3470,
  "duration": 5.09,
  "title": "Xe tải mất lái đâm dải phân cách cao tốc",
  "action": "Xe tải chạy tốc độ cao đâm rào chắn, nắp capo vàng bung lên hất vỡ kính",
  "category": "Traffic Accident",
  "mood": "shocking",
  "tags": ["highway", "truck", "dashcam"]
}
```
> Khi import vào **Premiere Pro MCP**: File vật lý được import là `fail_001.mp4`, nhưng MCP sẽ gán Clip Display Name trên Timeline hoặc Marker bằng `title` ("Xe tải mất lái..."). Người dựng phim nhìn vào Timeline vẫn thấy ngay tên tiếng Việt rõ ràng mà file trên ổ đĩa không sợ bị lỗi font hay đường dẫn quá dài!

---

## 🧹 Chính sách Tự động Dọn dẹp & Chống Tràn Ổ Đĩa (Auto-Purge Policy)

> [!CAUTION]
> **Không để file tạm tích tụ làm đầy ổ cứng!** Quy tắc tự động dọn rác nghiêm ngặt:

1. **Tự động xóa sau từng Phase (Post-Phase Auto-Purge):**
   * **Bảng Vision Sheet & Frames tạm:** Ngay sau khi kết thúc Phase 2 (User đã duyệt preview và chốt `scene_db.json`) $\rightarrow$ **Tự động xóa ngay lập tức 100% các ảnh Vision Sheet và frame trích xuất tạm thời**. Không lưu trữ ảnh thừa, chỉ giữ lại file text `scene_db.json` siêu nhẹ (~300 KB).
   * **File Proxy 480p:** Sau khi hoàn thành Phase 3 (Split xong các clip thật) $\rightarrow$ Agent chủ động hỏi User: *"Đã split xong video, bạn có muốn xóa file proxy_480p.mp4 để giải phóng ~900 MB dung lượng không?"* (hoặc tự động xóa nếu User bật chế độ tiết kiệm dung lượng).

2. **Giới hạn trần Cache (Max Cap 3 GB & TTL 48h):**
   * Thư mục cache tạm có hạn mức tối đa **3 GB**.
   * File tạm có thời gian sống tối đa **48 giờ (TTL = 48h)**. Nếu dung lượng chạm ngưỡng 3 GB, tự động xóa các file tạm cũ nhất (cơ chế FIFO) để ổ cứng không bao giờ bị đầy.

3. **Lệnh một chạm dọn sạch:**
   * Bất cứ lúc nào User gõ *"dọn dẹp"*, *"xóa rác"*, hoặc *"clean cache"*, Agent sẽ quét và xóa sạch 100% mọi file tạm, proxy và thumbnail phát sinh trong phiên làm việc chỉ trong 1 giây.

---

## Phase 1 — Tải video & Tạo Proxy Siêu Tốc

> ⚠️ Trước khi tải, Agent **phải hỏi User** về Thư mục lưu và Tên file/thư mục mong muốn!

### 1. Tải video YouTube
```bash
yt-dlp -f "bestvideo[height<=1080][ext=mp4]+bestaudio[ext=m4a]/best[height<=1080]" \
  --merge-output-format mp4 \
  --write-info-json \
  -o "<USER_CHOSEN_DIR>/<USER_CHOSEN_FILENAME>.%(ext)s" \
  "<URL>"
```

### 2. Tạo Proxy 480p H.264 Siêu Tốc
Video gốc thường dùng codec AV1 hoặc VP9, giải mã tuần tự rất chậm. Bắt buộc tạo proxy H.264 480p không âm thanh để tăng tốc bóc tách frame:

```bash
ffmpeg -y -i original_video.mp4 -vf "scale=-2:480" \
  -c:v libx264 -crf 28 -preset ultrafast -an proxy_480p.mp4
```
*Hiệu năng thực tế:* Video 68 phút tạo proxy chỉ mất 1 phút 21 giây.

---

## Phase 2 — Dò Cảnh Bằng Vision Sheet & Xử Lý Trùng Lặp Intro

### 1. Quét frame tuần tự siêu tốc kết hợp Check Intro Trực Tiếp (`scripts/frame_by_frame_detector.py`)
Quét **100% từng khung hình một** của file proxy trực tiếp qua OpenCV & NumPy:
* **Tốc độ:** ~3.280 frames/giây (quét toàn bộ 68 phút / 98.248 frames trong **29.8 giây**).
* **Vùng trung tâm kích hoạt (Active Center Crop):** Cắt lấy 50% chiều rộng ở giữa (`[w*0.25 : w*0.75]`) để triệt tiêu hoàn toàn viền đen 9:16 (tránh lỗi bỏ sót cắt do pha loãng màu).
* **Chỉ số kết hợp (Composite Metric):** Đo đồng thời độ lệch sáng vùng giữa ($\Delta \text{Luma} \ge 18$) và độ tương quan bảng màu ($\text{HSV Correlation} < 0.65$).
* **Lưu giữ `frame_info` & Keyframes (`frames_cache/`):** Tự động lưu lại thông tin khung hình đại diện (`mid_frame`, thời gian, độ sáng) và bộ đặc trưng điểm ảnh (ORB descriptors) của từng phân cảnh để kiểm tra, đối chiếu trực quan và làm bằng chứng xác thực.
* **So khớp Intro On-the-fly (Ngay trong Phase Detect):**
  - **0s – 15s:** Các phân cảnh Teaser Intro được nhận diện và lưu vào `intro_buffer` cùng keyframe và đặc trưng ORB (vùng trung tâm loại trừ chữ to).
  - **15s trở đi:** Mỗi khi phát hiện điểm cắt của một phân cảnh mới trong thân video, detector lập tức đối chiếu ngay khung hình đại diện với `intro_buffer`.
  - Sử dụng **ORB Feature Inliers (>20 inliers)** thay vì chỉ dựa vào màu sắc đơn thuần, triệt tiêu 100% tình trạng nhận diện sai (false positives).

### 2. Sinh Vision Sheet Mật Độ Cao (`scripts/vision_sheet_engine.py`)
Không lấy mẫu thưa 1 frame/giây (vốn dễ bỏ sót các pha hành động hay cú cắt micro-cut < 0.5s), skill hỗ trợ 3 chế độ lấy mẫu dày đặc:
* **Chế độ FULL FPS (1:1 — `step=1`, 24 frames/giây):** Trích xuất **100% từng frame một liên tục** (lưới 8×8 = 64 frames = ~2.67s/sheet). Dùng để soi kính hiển vi điểm tiếp giáp cắt cảnh và các chớp flash 1–2 frame.
* **Chế độ 1/2 FPS (`step=2`, ~12 frames/giây):** Lấy mẫu cách 2 frame (lưới 8×8 = ~5.33s/sheet). Dùng để soi trọn vẹn các pha va chạm tốc độ cao (xe đâm lan can nắp capo bay, xe lật, ngã thang).
* **Chế độ 1/5 FPS (`step=5`, ~4.8 frames/giây):** Lấy mẫu cách 5 frame (lưới 8×8 = ~13.3s/sheet). Dùng để soi mạch truyện (narrative arc: chuẩn bị ➔ biến cố ➔ hậu quả) và các teaser montage ngắn.

### 3. Nhận diện & Xử lý Trùng lặp Đoạn Intro / Teaser Montage
* **Hiện tượng:** Đoạn đầu (5s – 15s) thường là Teaser montage cắt nhanh các clip hấp dẫn nhất, bị chèn chữ to (*"HELLO EVERYONE"*, *"IF YOU NEED"*, *"A GOOD LAUGH TODAY"*), bị cắt cộc trước khi có kết quả. Các clip này thực chất **sẽ xuất hiện lại đầy đủ và sạch sẽ ở phần thân video**.
* **Quy trình xử lý chuẩn xác:**
  1. **Đối chiếu bằng Frame Info & ORB Inliers:** So khớp cấu trúc hình ảnh thực tế giữa các mẩu clip ở đoạn Intro với các clip đầy đủ phía sau (đạt >20–120 inliers đặc trưng).
  2. **Preview bảng đối chiếu trực quan (Side-by-Side Verification):**
     - Dùng các keyframe đã lưu trong `frames_cache/` để dựng bảng so sánh 2 ảnh gốc cạnh nhau (ảnh Intro dính chữ vs ảnh Bản gốc sạch) gửi cho User kiểm duyệt.
  3. **Hành động sau khi User xác nhận:**
     - **Replace & Promote:** Đôn các clip đầy đủ ở phía sau lên trên đầu danh sách (đổi tên, đặt số thứ tự ưu tiên lên trước `fail_001.mp4`, `fail_002.mp4`...).
     - **Xóa / Bỏ các clip Intro:** Loại bỏ hoàn toàn các mẩu teaser bị dính chữ và banner cảnh báo (WARNING), không xuất các file rác này.

### 4. Gộp Cảnh Bằng Tính Liên Tục Môi Trường & SSIM (Anti-Over-Split Aggregator)
Để chấm dứt tình trạng một tình huống fail bị xé vụn thành 3–5 clip nhỏ:
* **Gộp theo tính liên tục môi trường (Background & Color Palette Continuity):**
  Tự động gộp các shot liền kề có cùng bối cảnh, cùng bảng màu (HSV Corr $\ge 0.68$) hoặc cùng cấu trúc không gian (nhà xưởng, xe cộ, phòng gym, mặt nước...) thành 1 clip trọn vẹn.
* **Chống chém đôi khi va chạm (Anti-Impact Splitting):**
  Loại bỏ việc kích hoạt cắt khi có chớp sáng, túi khí bung, khói bụi hay tia lửa nếu khung cảnh trước và sau va chạm vẫn thuộc cùng một vụ việc.
* **Đo độ tương đồng cấu trúc (SSIM Verification):**
  Sử dụng chỉ số SSIM (`skimage.metrics.structural_similarity`) đối chiếu trực tiếp giữa khung hình trước và sau cú cắt. Nếu $\text{SSIM} > 0.20$ hoặc $(\text{SSIM} > 0.15 \text{ và } \text{HSV Corr} > 0.68)$, tự động hủy lệnh cắt và gộp thành 1 tình huống duy nhất.

### 5. Kiểm tra & Khử Trùng Lặp Source Nội Bộ và Liên N Video (`scripts/duplicate_source_detector.py`)
Khi xào dựng nhiều video hoặc xử lý các video tổng hợp compilation dài:
* **Trùng lặp nội bộ (Intra-Video Deduplication):**
  - Quét kiểm tra toàn bộ các clip trong cùng 1 video bằng Perceptual Hash (pHash).
  - Phát hiện các tình huống bị chèn lặp lại 2 lần ở các mốc thời gian khác nhau trong cùng video $\rightarrow$ Đánh dấu cảnh báo trên Preview Widget và tự động loại bỏ mẩu thừa.
* **Trùng lặp liên N video (Cross-Video / Multi-Video Deduplication):**
  - Nạp cơ sở dữ liệu vân tay số (Visual Fingerprint Index) của tất cả $N$ video trong dự án.
  - Tự động phát hiện các clip ở Video B đã từng xuất hiện ở Video A (ví dụ: phát hiện 41 clip của Video #87 tái sử dụng lại từ Video #86).
  - **Tùy chọn xử lý cho User:**
    1. *Bỏ qua không cắt:* Tiết kiệm dung lượng ổ cứng.
    2. *Vẫn cắt nhưng gắn thẻ:* Đánh dấu `is_duplicate_of: "fail_86_004.mp4"` vào `scene_db.json` để khi dựng timeline Premiere Pro MCP sẽ tự động lọc bỏ, không bao giờ dùng lại clip trùng trên cùng một timeline remix.

### 6. Hiểu Ngữ Cảnh Âm Thanh & Bóc Tách 6 Nhóm Âm Thanh (`scripts/audio_context_engine.py`)
Sử dụng kết hợp `librosa` (xử lý tín hiệu âm thanh) và `faster_whisper` (nhận diện giọng nói) để hiểu trọn vẹn ngữ cảnh âm thanh của từng clip:

#### A. Phân loại chuẩn hóa 6 nhóm âm thanh (6-Class Audio Taxonomy):
Mỗi phân cảnh trong `scene_db.json` được gán chính xác một trong 6 nhóm âm thanh:
1. `voice_nguoi_that` (Real Human Voice): Giọng nói người thật, tiếng kêu cảm thán (*"Oh shit!"*, *"Watch out!"*, *"Throw it!"*), tiếng cười đùa, đối thoại tự nhiên. Đặc trưng: độ biến thiên cao độ $F_0$ rộng ($\sigma(F_0) > 40\text{ Hz}$), micro-jitter tự nhiên và âm hưởng môi trường thực.
2. `voice_ai` (AI / Synthetic Voiceover): Giọng đọc trí tuệ nhân tạo (TikTok TTS, ElevenLabs, Siri, Google TTS). Đặc trưng: cao độ $F_0$ phẳng hoặc chuyển biến theo khuôn mẫu đều đặn ($\sigma(F_0) < 28\text{ Hz}$), nhịp độ từ ngữ cơ học, không có tạp âm sinh học hay hơi thở phòng.
3. `khong_am_thanh` (Mute / Complete Silence): Track âm thanh bị tắt hoàn toàn, hoặc âm lượng cực nhỏ dưới ngưỡng nghe ($\text{Peak} \le -42\text{ dB}$, $\text{RMS} < 0.005$).
4. `co_am_thanh_sfx` (SFX / Foley Only): Âm thanh va chạm, tiếng động cơ, tiếng phanh xe, tiếng rơi vỡ, thud, bước chân mà **KHÔNG có nhạc nền (BGM)** và **KHÔNG có giọng nói**. Đặc trưng: năng lượng bộ gõ/xung kích cao ($\text{Percussive Ratio} \ge 0.35$ hoặc $\text{Dynamic Range} \ge 2.5$).
5. `chi_co_nhac` (Music / BGM Only): Chỉ có nhạc nền, giai điệu hoặc tiết tấu âm nhạc mà **KHÔNG có lời thoại hay tiếng va chạm nổ lớn**. Đặc trưng: tỷ lệ hòa âm cao ($\text{Harmonic Ratio} \ge 0.40$), nhịp điệu đều đặn ($\text{Tempo } 60-200\text{ BPM}$).
6. `hon_hop` (Mixed Tracks): Hỗn hợp nhiều lớp âm thanh cùng xuất hiện (Nhạc BGM + Giọng nói bình luận, Nhạc BGM + Cú đâm va chạm lớn, hoặc Giọng nói la hét trong lúc va chạm mạnh).

#### B. Audio-Assisted Zero-Cut Snapping (Bảo vệ âm thanh & Khử Click/Pop khi cắt):
Khi cắt video, việc cắt cứng tại ranh giới thị giác ($T_{\text{visual}}$) thường làm chém ngang một câu thoại dở dang hoặc cắt cụt tiếng ngân (reverb) của cú va chạm, đồng thời gây tiếng nổ (click/pop) do lệch điện áp DC.
* **Cơ chế hoạt động:**
  1. Quét biên độ sóng âm trong cửa sổ $\pm 0.25\text{s}$ xung quanh điểm cắt thị giác ($\pm 4-6$ video frames).
  2. Dò tìm điểm trũng năng lượng (Local RMS Energy Minimum Dip).
  3. Snap chính xác vào điểm sóng âm đi qua điện áp 0 (**Zero-Crossing**: $y[i] \cdot y[i+1] \le 0$).
  4. Bảo vệ trọn vẹn âm cuối của lời thoại hoặc đuôi vang của cú nổ trước khi chuyển cảnh.

#### C. Dò Điểm Va Chạm / Cao Trào (Audio Climax / Peak Impact Detection):
* Quét năng lượng xung kích (Onset Strength & RMS Spike) để xác định chính xác đến từng mili-giây thời điểm xảy ra va chạm mạnh nhất (`peak_impact_time_sec` và `peak_impact_frame`).
* Premiere Pro MCP tự động cắm **Impact Marker màu đỏ** trên Timeline đúng nhịp va chạm để chèn SFX hay camera shake.

### 7. Ghi nhận ngữ cảnh vào cơ sở dữ liệu (`scene_db.json`)
Agent phân tích trực quan & âm thanh và điền các trường ngữ nghĩa cho từng tình huống hoàn chỉnh:
* `title`: Tên súc tích của pha fail/clip (VD: *"Xe tải mất lái đâm dải phân cách cao tốc"*).
* `action`: Diễn biến chi tiết sự việc.
* `category`: Phân loại (Traffic, Workplace, Gym, Water, Home Repair, Extreme Weather...).
* `mood`: Tông cảm xúc (funny, shocking, chaotic, clumsy...).
* `tags`: Từ khóa tìm kiếm để dựng timeline.
* `start_f`, `end_f`, `duration`: Khung hình và thời gian chính xác (số nguyên).
* `shots_merged`: Số lượng cú máy con / pha va chạm đã được gom lại thành công.
* `duplicate_info`: Thông tin clip trùng lặp nội bộ hoặc liên video (nếu có).
* `audio_context`: Dữ liệu âm thanh thông minh:
  - `audio_type`: 1 trong 6 loại (`voice_nguoi_that`, `voice_ai`, `khong_am_thanh`, `co_am_thanh_sfx`, `chi_co_nhac`, `hon_hop`).
  - `has_speech`, `speech_text`, `speech_language`.
  - `sound_class`, `mood`.
  - `peak_impact_time_sec`, `peak_impact_frame_relative`.
  - `loudness_peak_db`, `dynamic_range`, `harmonic_ratio`, `percussive_ratio`.

---

## 🛑 Step 2.5 (Bắt buộc) — Preview Widget & Human-in-the-Loop Approval

> [!IMPORTANT]
> **Tuyệt đối KHÔNG ĐƯỢC tự ý chạy lệnh cắt (Phase 3 Split) ngay sau khi vừa detect xong!**
> Hệ thống bắt buộc phải dừng lại và thực thi quy trình Human-in-the-Loop:
>
> 1. **Dựng Preview Widget Trực Quan:** Tạo file HTML widget tương tác (hoặc Carousel ảnh đối chiếu) lấy dữ liệu từ `frames_cache/`:
>    * Hiển thị danh sách từng tình huống hoàn chỉnh (Story Event).
>    * Ảnh thumbnail / keyframe thực tế trích xuất từ video.
>    * Thời gian bắt đầu $\rightarrow$ kết thúc, độ dài từng vụ việc.
>    * Badge hiển thị số lượng cú máy con / pha nổ đã được gộp (`Đã gộp X góc máy`).
>
> 2. **Chờ User Phê Duyệt (Human Confirmation):**
>    * Trình bày bảng xem trước và dừng lại hỏi User: *"Bạn đã duyệt danh sách các tình huống được gộp ở trên chưa? Có cần tinh chỉnh ranh giới điểm cắt nào không trước khi tiến hành cắt thật?"*
>    * Chỉ khi User xác nhận đồng ý (`ok`, `tiến hành cắt`, `proceed`), Agent mới được phép kích hoạt Phase 3.

---

## Phase 3 — Frame-Accurate Splitting (`-vframes`)

> 💡 Ở Phase 3, Agent **chỉ việc đọc trực tiếp danh sách từ `scene_db.json`** (nơi đã có sẵn `file_name` và `file_path` chuẩn chỉnh mà User đã duyệt và chốt ở Phase 2). Không cần hỏi lại, không sợ nhầm lẫn tên file!

Khi xuất video clip thật từ video gốc 1080p, **tuyệt đối không dùng `-to <timecode>`** vì thuật toán làm tròn của FFmpeg có thể kéo theo 1 frame của cảnh kế tiếp.

### Cú pháp cắt chuẩn xác 100% không dính đuôi:
```bash
# Đọc file_path từ scene_db.json và chỉ định số khung hình chính xác qua -vframes:
ffmpeg -y -ss <START_SECONDS> -i original_video.mp4 \
  -vframes <TOTAL_FRAMES> \
  -c:v libx264 -crf 18 -preset ultrafast -c:a aac \
  "<file_path_from_scene_db>"
```

*Công thức tính:*
* $\text{START\_SECONDS} = \text{start\_f} / \text{FPS}$
* $\text{TOTAL\_FRAMES} = \text{end\_f} - \text{start\_f}$

*Kiểm chứng thực tế:* Frame cuối cùng của video xuất ra 100% thuộc về nội dung cảnh hiện tại, không dính 1 pixel nào của cảnh sau.

---

## Phase 4 — Premiere Pro MCP Bridge

Sau khi các clip sạch được xuất theo đúng yêu cầu đặt tên của User:
1. **Lọc kịch bản thông minh:** Lọc các clip theo `category`, `mood`, `tags`, và **tự động loại bỏ các clip bị trùng lặp liên video (`duplicate_info`)**.
2. **Gọi MCP Tools để dựng Timeline:**
   * `createSequence`: Tạo sequence theo độ phân giải và fps mong muốn.
   * `importFiles`: Import các clips đã cắt vào Project Bin và đặt tên clip bằng `title`.
   * `insertClipToTimeline`: Xếp các clips lên timeline theo đúng thứ tự kịch bản remix.
   * **`addMarker` (Audio Climax Marker):** Đọc trường `audio_context.peak_impact_frame` để tự động cắm **Marker màu đỏ (Impact)** ngay tại khung hình xảy ra va chạm/đỉnh điểm, giúp editor ghép sound effect hay hiệu ứng slow-motion/shake màn hình chuẩn xác từng nhịp.
   * **Tự động gắn phụ đề:** Đọc trường `audio_context.speech_text` để tạo phụ đề / chú thích tự động trên timeline.
   * `addTransition`: Thêm chuyển cảnh mượt mà giữa các clip.
