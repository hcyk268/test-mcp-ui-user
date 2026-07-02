# Figma-to-Web UI Conversion Agent Rules

## Mục tiêu
Chuyển đổi chính xác 100% từ một link frame Figma thành file HTML/CSS web, đảm bảo pixel-perfect với thiết kế gốc.

---

## 1. Kỹ năng phân tích Figma (Figma Analysis)

### 1.1. Lấy thông tin từ link Figma
- **Trích xuất `file_key`**: Từ URL Figma `https://www.figma.com/design/{file_key}/...`, trích xuất đoạn ID file.
- **Trích xuất `node_id`**: Từ parameter `node-id=XXX-YYY` trong URL, chuyển đổi thành format `XXX:YYY` (thay dấu `-` bằng `:`).

### 1.2. Chụp ảnh thiết kế và lưu về local (Screenshot & Save)
- Sử dụng MCP tool `get_screenshot` với `file_key` và `node_id` để lấy ảnh tổng quan frame.
- Luôn set `scale: 2` để có ảnh rõ nét, dễ phân tích chi tiết nhỏ.
- **BẮT BUỘC lưu ảnh screenshot về thư mục `assets/figma_ref/`** để so sánh sau:
  - Dùng MCP tool `save_screenshots` để tải ảnh về local.
  - Hoặc copy ảnh từ output `get_screenshot` vào thư mục dự án.
  - Đặt tên file theo format: `figma_[page-name]_[node-id].png` (VD: `figma_hotel-detail_168-167.png`).
- Ảnh Figma này sẽ dùng làm **reference gốc** để so sánh pixel-by-pixel với kết quả browser.

### 1.3. Đọc cấu trúc node tree (Node Inspection)
- Sử dụng `get_node` với `node_id` để lấy cấu trúc JSON đầy đủ của frame.
- Phân tích cây node từ trên xuống: Frame → Section → Group → Component → Text/Shape.
- Chú ý các thuộc tính quan trọng:
  - **`absoluteBoundingBox`**: Vị trí x, y, width, height chính xác.
  - **`fills`**: Màu nền, gradient, ảnh.
  - **`strokes`**: Border color, width, style.
  - **`effects`**: Shadow (drop shadow, inner shadow), blur.
  - **`cornerRadius`**: Bo góc.
  - **`style`** (cho text): fontFamily, fontSize, fontWeight, lineHeight, letterSpacing, textAlignHorizontal.
  - **`characters`** (cho text): Nội dung text thực tế.
  - **`layoutMode`**: Auto-layout direction (HORIZONTAL / VERTICAL).
  - **`itemSpacing`**: Gap giữa các item trong auto-layout.
  - **`paddingLeft/Right/Top/Bottom`**: Padding của frame.

### 1.3.1. Bắt buộc phân tích tới node/layer cực nhỏ
- **Không dừng ở mức frame lớn hoặc component lớn**. Phải tiếp tục drill-down cho đến khi xác định rõ từng phần tử nhỏ:
  - Background nền riêng.
  - Ảnh riêng (`image fill`, `RECTANGLE` có ảnh, `INSTANCE` chứa ảnh).
  - Text riêng.
  - Icon/vector riêng.
  - Shadow/blur/gradient riêng.
  - Divider, badge, chip, stroke, overlay riêng.
- Với mọi khu vực quan trọng của UI, phải trả lời được:
  1. **Đây là ảnh thật hay chỉ là shape/vector?**
  2. **Đây là text thật hay text đã nằm chết trong ảnh?**
  3. **Đây là icon riêng hay là một phần của ảnh export?**
  4. **Đây có phải decorative layer có thể dựng bằng CSS thay vì export bitmap không?**
- Nếu một node cha chứa nhiều loại layer trộn lẫn, phải bóc tiếp children cho đến khi có thể map từng phần sang HTML/CSS riêng biệt.

### 1.3.2. Quy tắc map node → implementation
- Mỗi node nhỏ phải được quyết định theo một trong 4 hướng:
  1. **HTML text thật**.
  2. **SVG/vector thật**.
  3. **CSS shape/effect**.
  4. **Bitmap asset riêng lẻ**.
- **Không được dùng một ảnh export lớn để “ôm” luôn text, icon, badge, overlay, hoặc nhiều card con**, trừ khi chính thiết kế gốc xác nhận đó là một ảnh duy nhất không thể tách.
- Nếu export asset, phải export ở **mức node nhỏ nhất hợp lý**, ví dụ:
  - 1 card ảnh riêng.
  - 1 ảnh thumbnail riêng.
  - 1 icon SVG riêng.
  - 1 texture/background layer riêng.
- Trước khi code, nên lập mapping ngắn theo kiểu:
  - `node A` → background CSS.
  - `node B` → `<img>`.
  - `node C` → `<h2>`.
  - `node D` → inline SVG.
  - `node E` → shadow bằng CSS.

### 1.4. Quét nhanh tất cả text nodes
- Sử dụng `scan_text_nodes` để lấy toàn bộ nội dung text trong frame.
- Đảm bảo không bỏ sót bất kỳ text nào khi dựng HTML.

### 1.5. Tìm kiếm node theo tên
- Sử dụng `search_nodes` để tìm các component cụ thể (button, card, input...).
- Sử dụng `scan_nodes_by_types` để quét theo loại (TEXT, RECTANGLE, FRAME, INSTANCE...).

### 1.6. Lấy thông tin về fonts, styles, variables
- Sử dụng `get_fonts` để biết font nào đang dùng → import vào HTML.
- Sử dụng `get_styles` để lấy color styles, text styles, effect styles đã định nghĩa.
- Sử dụng `get_local_components` để hiểu component library.

---

## 2. Kỹ năng đo đạc và tính toán (Measurement & Calculation)

### 2.1. Tính toán kích thước
- Dùng `absoluteBoundingBox` để tính:
  - **Width/Height chính xác** của mỗi element.
  - **Gap** giữa các element = `element2.x - (element1.x + element1.width)`.
  - **Padding** = khoảng cách từ child đến parent edge.
  - **Margin** = khoảng cách giữa sibling elements.

### 2.2. Chuyển đổi màu sắc
- Figma trả về màu dạng RGBA với giá trị 0-1. Chuyển đổi:
  - `r * 255`, `g * 255`, `b * 255` → hex color.
  - Opacity < 1 → dùng `rgba()`.
- Gradient: Chuyển đổi gradient stops từ Figma sang CSS `linear-gradient()` hoặc `radial-gradient()`.

### 2.3. Chuyển đổi typography
- `fontSize` → `font-size` (px).
- `fontWeight` → `font-weight` (100-900).
- `lineHeightPx` → `line-height` (px hoặc unitless ratio).
- `letterSpacing` → `letter-spacing` (px hoặc em).
- `textAlignHorizontal` → `text-align` (LEFT→left, CENTER→center, RIGHT→right, JUSTIFIED→justify).
- `textDecoration` → `text-decoration`.
- `textCase` → `text-transform` (UPPER→uppercase, LOWER→lowercase, TITLE→capitalize).

### 2.4. Chuyển đổi effects (Shadow, Blur)
- **Drop Shadow**: `box-shadow: offsetX offsetY blur spread color`.
  - `offset.x` → offsetX, `offset.y` → offsetY.
  - `radius` → blur radius.
  - `spread` → spread radius.
  - Color: chuyển đổi RGBA.
- **Inner Shadow**: Thêm `inset` vào `box-shadow`.
- **Layer Blur**: `filter: blur(radius)`.
- **Background Blur**: `backdrop-filter: blur(radius)`.

---

## 3. Kỹ năng dựng HTML cấu trúc (HTML Structuring)

### 3.1. Mapping Figma → HTML semantic tags
| Figma Element | HTML Tag |
|---|---|
| Frame (page-level) | `<main>`, `<section>` |
| Frame (container) | `<div>` |
| Frame (navigation) | `<nav>`, `<header>` |
| Frame (card) | `<article>` |
| Frame (footer) | `<footer>` |
| Text (heading) | `<h1>`-`<h6>` |
| Text (paragraph) | `<p>`, `<span>` |
| Text (link) | `<a>` |
| Rectangle (button) | `<button>`, `<a>` |
| Image fill | `<img>` |
| Input field | `<input>`, `<select>`, `<textarea>` |
| List items | `<ul>`, `<ol>`, `<li>` |
| Form container | `<form>` |
| Separator line | `<hr>` |

### 3.2. Đặt ID và class
- Mỗi element tương tác phải có **ID duy nhất** (`id="btn-submit-booking"`).
- Đặt tên class theo BEM hoặc semantic: `.tour-card`, `.price-wrap`, `.btn-primary`.
- Nhóm các element cùng loại dưới class chung.

### 3.3. Thứ tự xây dựng
1. **Header** (logo, nav, actions) → luôn làm trước.
2. **Main content** (hero/banner → content sections).
3. **Footer** (brand, links, contact).
4. SVG icons inline thay vì icon fonts để đảm bảo chính xác.

---

## 4. Kỹ năng viết CSS chính xác (Pixel-Perfect CSS)

### 4.1. Layout System
- **Figma Auto-Layout HORIZONTAL** → CSS `display: flex; flex-direction: row;`
- **Figma Auto-Layout VERTICAL** → CSS `display: flex; flex-direction: column;`
- **Figma Grid layout** → CSS `display: grid; grid-template-columns: ...;`
- `itemSpacing` → `gap: Xpx;`
- `primaryAxisAlignItems`:
  - MIN → `justify-content: flex-start`
  - CENTER → `justify-content: center`
  - MAX → `justify-content: flex-end`
  - SPACE_BETWEEN → `justify-content: space-between`
- `counterAxisAlignItems`:
  - MIN → `align-items: flex-start`
  - CENTER → `align-items: center`
  - MAX → `align-items: flex-end`

### 4.2. Sizing
- Dùng **fixed pixel values** cho layout chính xác (ví dụ: `width: 368px; height: 192px;`).
- Container chính dùng `max-width` + `margin: 0 auto` để center.
- Ảnh luôn set `object-fit: cover` và `width: 100%; height: 100%;` trong container fixed-size.

### 4.3. CSS Variables
Định nghĩa design tokens ở `:root`:
```css
:root {
  --primary-color: #c8102e;
  --text-dark: #1b1c1c;
  --text-muted: #5f5e5e;
  --border-color: #e5bdbb;
  --bg-light: #f5f3f3;
  --font-sans: 'Be Vietnam Pro', sans-serif;
  --transition-smooth: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}
```

### 4.4. Quy tắc CSS quan trọng
- **Không dùng `!important`** trừ trường hợp bất khả kháng.
- **Không dùng inline style** trong HTML — tất cả style phải ở file CSS riêng.
- **Tên class phải nhất quán** giữa HTML và CSS — đây là nguyên nhân lỗi #1 khi class trong HTML không khớp với selector trong CSS.
- Tổ chức CSS theo sections với comment rõ ràng:
  ```css
  /* --- 1. GLOBAL RESET & VARIABLES --- */
  /* --- 2. HEADER STYLES --- */
  /* --- 3. HERO SECTION --- */
  ```

### 4.5. Hover & Interaction States
- Mọi button và link phải có `:hover` state.
- Sử dụng `transition` cho smooth animations.
- Cards nên có `transform: translateY(-4px)` + `box-shadow` tăng khi hover.

---

## 5. Kỹ năng xử lý Assets (Image & Icon Handling)

### 5.1. Ảnh
- Nếu Figma có ảnh fill, cần:
  1. Export ảnh từ Figma hoặc dùng placeholder.
  2. Đặt trong thư mục `assets/`.
  3. Set `object-fit: cover` để ảnh luôn fill đúng container.
- Ảnh cần `alt` text mô tả nội dung.
- **Chỉ export đúng phần ảnh cần thiết**, không export cả cụm UI nếu trong đó còn chứa text, icon, badge, hoặc nhiều layer có thể dựng được bằng HTML/CSS.
- Ví dụ đúng:
  - Export riêng ảnh card Hạ Long.
  - Export riêng ảnh card Hội An.
  - Export riêng ảnh card Sa Pa.
- Ví dụ sai:
  - Export nguyên mảng hero/trái/phải mà trong ảnh đã dính luôn text, icon, gradient overlay và nhiều card con.

### 5.1.1. Quy tắc “node nhỏ nhất hợp lý”
- Trước khi export bất kỳ bitmap nào, phải hỏi:
  - Có thể dựng phần này bằng CSS không?
  - Có thể giữ text ở HTML không?
  - Có thể giữ icon ở SVG không?
  - Có thể tách card/image con riêng không?
- Nếu câu trả lời là “có”, **phải tách** thay vì export cả khối.
- Mục tiêu là asset export chỉ chứa **pixel content thực sự cần rasterize**, không chứa semantic content.

### 5.2. SVG Icons
- Lấy SVG path data từ Figma node inspection.
- Inline SVG trong HTML để dễ style bằng CSS (`fill`, `stroke`, `color`).
- Set `viewBox` đúng theo kích thước gốc.
- Dùng `currentColor` cho fill/stroke để icon inherit color từ parent.

### 5.3. Logo
- Logo phải là SVG hoặc PNG chất lượng cao.
- Cần 2 version: logo màu (cho header) và logo trắng (cho footer dark background).

---

## 6. Kỹ năng kiểm tra và so sánh (Verification)

### 6.1. Browser Preview
- Sau khi dựng xong, **bắt buộc** mở file HTML trên browser để kiểm tra.
- Dùng local HTTP server (`python -m http.server 8000` hoặc tương tự).
- Kiểm tra trang trên viewport width 1536px (desktop standard).

### 6.2. So sánh pixel-by-pixel (Side-by-Side Comparison)
- **Bước 1**: Lấy lại ảnh Figma đã lưu từ `assets/figma_ref/` (hoặc chụp mới bằng `get_screenshot`).
- **Bước 2**: Dùng browser subagent chụp screenshot trang web đã dựng.
- **Bước 3**: Đặt 2 ảnh cạnh nhau (Figma vs Browser) và so sánh từng vùng:
  - **Header**: Logo, nav links, action buttons.
  - **Main content**: Cards, images, text blocks, sidebar.
  - **Footer**: Columns, links, social icons.
- **Bước 4**: Ghi lại các điểm sai lệch và fix.
- **Bước 5**: Lặp lại cho đến khi 2 ảnh giống nhau.

- Kiểm tra các điểm thường sai:
  1. **Font size / weight** — dễ bị sai 1-2px.
  2. **Spacing (gap, padding, margin)** — đo lại nếu trông khác.
  3. **Border radius** — đặc biệt ở cards và buttons.
  4. **Colors** — kiểm tra hex code chính xác.
  5. **Shadow** — thường bị thiếu hoặc sai opacity.
  6. **Text content** — copy chính xác, không viết tắt.
  7. **Sai phương pháp dựng** — nếu browser “giống ảnh” nhưng đang dùng ảnh ghép chứa cả text/icon/UI thì vẫn xem là **chưa đạt**.

### 6.2.1. Kiểm tra đúng phương pháp, không chỉ đúng hình
- Ngoài so sánh screenshot, phải kiểm tra lại DOM/CSS để xác nhận:
  - Text trong thiết kế đang là text HTML thật.
  - Icon đang là SVG/vector thật hoặc CSS shape.
  - Ảnh chỉ là ảnh, không gói chung text và UI.
  - Background, overlay, blur, shadow đã được dựng tách lớp hợp lý.
- Một màn hình nhìn giống Figma nhưng được làm bằng “ảnh chụp nguyên cụm” vẫn bị coi là **sai quy trình**.
- **Pixel-perfect không chỉ là giống bằng mắt, mà còn phải đúng cấu trúc layer**.

> **Tip**: Tạo artifact markdown với cả 2 ảnh embedded để user dễ so sánh trực quan.

### 6.3. Checklist kiểm tra cuối
- [ ] Header: Logo, navigation, action buttons đúng vị trí.
- [ ] Màu sắc: Tất cả colors match Figma.
- [ ] Typography: Font family, size, weight, line-height đúng.
- [ ] Spacing: Gap, padding, margin pixel-perfect.
- [ ] Images: Hiển thị đúng, cover container.
- [ ] Icons: SVG render đúng, đúng màu.
- [ ] Buttons: Style đúng, có hover effect.
- [ ] Footer: Layout đúng, links hoạt động.
- [ ] Responsive container: Center trên desktop.

---

## 7. Quy trình làm việc end-to-end (Full Workflow)

```
Step 1: Nhận link Figma
    ↓
Step 2: Trích xuất file_key + node_id
    ↓
Step 3: Chụp screenshot frame (get_screenshot, scale=2)
    ↓
Step 4: LƯU ảnh Figma về assets/figma_ref/ (save_screenshots)
    ↓
Step 5: Phân tích node tree (get_node, depth đủ sâu)
    ↓
Step 6: Quét text nodes (scan_text_nodes)
    ↓
Step 7: Lấy fonts + styles (get_fonts, get_styles)
    ↓
Step 8: Phân tích ảnh screenshot — xác định layout tổng thể:
         - Số columns
         - Header/Content/Footer structure
         - Card patterns
         - Color palette
    ↓
Step 9: Viết HTML structure (semantic tags, IDs, classes)
    ↓
Step 10: Viết CSS styles (layout → typography → colors → effects)
    ↓
Step 11: Thêm assets (images, SVG icons)
    ↓
Step 12: Mở browser, chụp screenshot browser
    ↓
Step 13: ĐẶT CẠNH NHAU: Figma screenshot vs Browser screenshot
    ↓
Step 14: So sánh từng vùng — ghi lại sai lệch — fix
    ↓
Step 15: Lặp lại Step 12-14 cho đến khi pixel-perfect
```

---

## 8. Lỗi thường gặp và cách tránh (Common Pitfalls)

### 8.1. CSS Class Name Mismatch ⚠️
> **Đây là lỗi NGHIÊM TRỌNG NHẤT và hay gặp nhất.**

- Khi tạo HTML, đặt class name `hotel-columns-grid`.
- Khi viết CSS, viết selector `.hotel-columns-container`.
- **Kết quả**: Layout hoàn toàn vỡ, element không có style.
- **Cách tránh**: 
  - Viết HTML trước, rồi copy chính xác class name sang CSS.
  - Sau khi viết xong, grep kiểm tra tất cả class trong HTML đều có trong CSS.
  - Khi tạo phiên bản guest từ trang chính, **COPY nguyên file rồi chỉ sửa header/links**, KHÔNG viết lại từ đầu.

### 8.2. Missing CSS Sections
- Khi file CSS lớn (>3000 lines), dễ quên thêm section mới.
- **Cách tránh**: Tổ chức CSS theo numbered sections, thêm section mới ở cuối file.

### 8.3. Footer Inconsistency
- Mỗi page dùng footer class khác nhau.
- **Cách tránh**: Copy footer HTML giống nhau cho tất cả pages, chỉ thay links nếu cần.

### 8.4. Figma Color Format
- Figma dùng RGBA 0-1, CSS dùng 0-255 hoặc hex.
- **Cách tránh**: Luôn chuyển đổi `Math.round(r * 255)` trước khi viết CSS.

### 8.5. Auto-Layout vs Absolute Position
- Figma có thể mix auto-layout và absolute positioning.
- **Cách tránh**: Ưu tiên dùng Flexbox/Grid. Chỉ dùng `position: absolute` cho overlays, badges, floating elements.

### 8.6. Export Cả Cụm UI Thành Một Ảnh ⚠️
> **Đây là lỗi quy trình nghiêm trọng, dù kết quả nhìn có vẻ giống Figma.**

- Sai điển hình:
  - Export nguyên hero visual đã chứa sẵn text + icon + nhiều card.
  - Export nguyên sidebar/card lớn thay vì tách từng ảnh con.
  - Export cả block auth/dashboard chỉ vì “nhanh hơn”.
- Hậu quả:
  - Mất semantic HTML.
  - Không responsive đúng.
  - Không chỉnh pixel-level từng thành phần được.
  - DOM không phản ánh đúng thiết kế Figma.
- **Cách tránh**:
  1. Luôn phân tích children của node lớn.
  2. Chỉ export asset ở node nhỏ nhất hợp lý.
  3. Giữ text/icon/badge/overlay ở HTML/CSS/SVG bất cứ khi nào có thể.
  4. Review lại từng asset export để chắc chắn nó không “dính” quá nhiều layer khác nhau.

---

## 9. Quy tắc đặc biệt cho dự án này (Project-Specific Rules)

### 9.1. Cấu trúc file
```
UI/
├── index.html          (Trang chủ - logged in)
├── index-guest.html    (Trang chủ - guest)
├── tour.html           (Tour listing - logged in)
├── tour-guest.html     (Tour listing - guest)
├── [page].html         (Logged in version)
├── [page]-guest.html   (Guest version)
├── index.css           (Single CSS file cho tất cả pages)
├── assets/             (Images, logos, icons)
│   ├── logo.svg
│   ├── logo_white.svg
│   └── *.jpg / *.png
```

### 9.2. Guest vs Logged-in Pages
- **Logged-in**: Header có avatar user.
- **Guest**: Header có notification bell + nút "Đăng nhập" đỏ.
- **Quy tắc tạo guest page**:
  1. Copy file logged-in.
  2. Thay header section (avatar → guest buttons).
  3. Thay TẤT CẢ links: `page.html` → `page-guest.html`.
  4. Kiểm tra breadcrumb, back buttons cũng trỏ đúng.

### 9.3. CSS Organization
```css
/* --- 1. GLOBAL RESET & VARIABLES --- */
/* --- 2. HEADER STYLES --- */
/* --- 3. HERO SECTION --- */
/* --- 4. SEARCH WIDGET --- */
/* --- 5. DESTINATIONS --- */
/* --- 6. TOUR LISTING --- */
/* --- 7. TOUR DETAIL --- */
/* --- 8. BOOKING PAGE --- */
/* --- 9. HOTEL LISTING --- */
/* --- 10. TRAIN BOOKING --- */
/* --- 11. FLIGHT BOOKING --- */
/* --- 12. TOUR DETAIL INNER --- */
/* --- 13. HOTEL DETAIL --- */
/* --- 14. FLIGHT DETAIL --- */
/* --- 15. TRAIN DETAIL --- */
/* --- 16. FOOTER --- */
```

### 9.4. Design Tokens
| Token | Value |
|---|---|
| Primary Red | `#c8102e` |
| Dark Red | `#9e001f` |
| Text Dark | `#1b1c1c` |
| Text Muted | `#5f5e5e` |
| Text Light | `#5c403f` |
| Border Pink | `#e5bdbb` |
| Border Grey | `#e2dfde` |
| BG Light | `#f5f3f3` |
| BG Cream | `#fbf9f8` |
| Star Yellow | `#eab308` |
| Success Green | `#16a34a` |
| Font | Be Vietnam Pro |
| Container Width | 1152px |
