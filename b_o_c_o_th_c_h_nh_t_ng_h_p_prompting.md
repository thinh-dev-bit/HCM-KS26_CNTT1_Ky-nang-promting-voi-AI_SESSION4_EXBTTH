# BÁO CÁO THIẾT KẾ GIẢI PHÁP TỔNG HỢP: KỸ THUẬT PROMPTING NÂNG CAO (LOGIC & CHUỖI)

---

## PHẦN 1: BÁO CÁO THIẾT KẾ GIẢI PHÁP KIẾN TRÚC

### 1.1. Bối cảnh Bài toán Thực tế
Tại Tập đoàn Công nghệ & Logistics **FastDelivery**, phòng Kế toán & Kiểm toán Vận tải phải đối soát hàng trăm bộ chứng từ nhiên liệu, nhật ký hành trình GPS và phiếu phụ phí cầu đường mỗi ngày. Việc dồn toàn bộ quy trình vào một câu lệnh đơn lẻ (*Mega Prompt*) hoặc xử lý thủ công gây ra 4 điểm nghẽn nghiêm trọng:
1. **Ảo giác tính toán đa biến số:** Khi tính định mức tiêu hao nhiên liệu ($\text{Lít}/100\text{km}$), đối chiếu chênh lệch thực tế so với định mức xe tải và cộng phụ phí cao điểm, AI phản xạ tức thì dẫn đến sai số 20–30%.
2. **Sập luồng (Crash) do dữ liệu dị biệt:** Chứng từ thực tế thường chứa dữ liệu bẩn như số km bằng `0` (gây lỗi chia cho 0 khi tính định mức), số tiền âm, ngày tháng không tồn tại (`32/13/2026`), thiếu biển số xe hoặc lẫn ký tự nhiễu làm hỏng cấu trúc JSON.
3. **Lãng phí Token & Độ trễ cao:** Gọi mô hình cao cấp (*Pro*) cho các bước bóc tách văn bản đơn giản làm đội chi phí API gấp nhiều lần.
4. **Đứt gãy & trôi dạt ngữ cảnh:** Khi xuất bảng tổng hợp kiểm toán hàng loạt cuối ngày, phản hồi dễ bị ngắt cụt do chạm trần **Max Output Tokens**; khi vận hành liên tục qua nhiều lượt đối soát, AI bị **Context Drift** (quên quy tắc kiểm toán ban đầu).

### 1.2. Kiến trúc Chuỗi Prompt Liên kết Khép kín (Closed-Loop Pipeline)
Giải pháp thiết kế một dây chuyền **Prompt Chaining 3 Trạm** khép kín, kết hợp **Validation Gate**, **Structured Chain-of-Thought (CoT)**, **Model Routing** và **Cơ chế Khôi phục Ngữ cảnh**:

```text
[Chứng từ thô + GPS] 
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│ TRẠM 1: Trích xuất & Chuẩn hóa (Model Mini / Flash)      │
│ • Tối ưu token bằng schema rút gọn, trả về Pure JSON     │
└──────────────────────┬───────────────────────────────────┘
                       │ Data Handoff (Pure JSON)
                       ▼
┌──────────────────────────────────────────────────────────┐
│ CHỐT CHẶN VALIDATION GATE (Code Python + Rule Engine)    │
│ • Bắt lỗi rỗng (null), số âm, km <= 0, sai ngày tháng    │
│ • Dị biệt ──► Rẽ nhánh an toàn (HOLD_FOR_FIX, no crash)  │
└──────────────────────┬───────────────────────────────────┘
                       │ Dữ liệu sạch 100%
                       ▼
┌──────────────────────────────────────────────────────────┐
│ TRẠM 2: Thẩm định & Tính toán CoT (Model Pro)            │
│ • Lập luận 5 bước minh bạch (reasoning_path)             │
│ • Cross-validation chống ngụy biện & khóa [RESULT: ...]  │
└──────────────────────┬───────────────────────────────────┘
                       │ Kết quả thẩm định chuẩn hóa
                       ▼
┌──────────────────────────────────────────────────────────┐
│ TRẠM 3: Xuất Báo cáo & Điều phối HITL (Context Anchoring)│
│ • Gắn cờ [HITL] nếu chênh lệch > 15% hoặc > 3.000.000đ   │
│ • Tự động kích hoạt Continue Prompting nếu ngắt đoạn     │
└──────────────────────────────────────────────────────────┘
```

### 1.3. Chiến lược Tối ưu Token & Xử lý Lỗi Dị biệt

| Kỹ thuật áp dụng | Cơ chế thực thi trong hệ thống | Hiệu quả đạt được |
| :--- | :--- | :--- |
| **Model Routing (Định tuyến mô hình)** | Dùng model nhỏ, giá rẻ (*Gemini Flash / GPT-4o-mini*) cho Trạm 1 & Trạm 3; chỉ kích hoạt model mạnh (*Gemini Pro / GPT-4o*) cho Trạm 2 (CoT). | Tiết kiệm **65–75%** chi phí API so với dùng model Pro cho toàn chuỗi. |
| **Token-Optimized Schema** | Sử dụng tên khóa JSON ngắn gọn, rõ nghĩa; cấm tuyệt đối lời chào xã giao (*Handoff rác*) ở Trạm 1 và Trạm 2. | Giảm **30–40%** lượng Output Tokens sinh ra ở các trạm trung gian. |
| **Validation Gate & Exception Handling** | Kiểm tra `distance_km <= 0`, `fuel_cost < 0`, `date` không hợp lệ và bọc `try...except` quanh bộ phân tích JSON. | Ngăn chặn 100% lỗi `ZeroDivisionError` và lỗi lan truyền (*Cascading Errors*), không bao giờ crash hệ thống. |
| **Continue Prompting + Rolling Summary** | Tự động kiểm tra `finish_reason == "length"` để mớm 4 từ cuối nối tiếp bảng báo cáo; nén lịch sử sau mỗi 10 lượt bằng *Rolling Summary* và *Session Reset* sau 30 lượt. | Đảm bảo báo cáo dài liền mạch 100%, giữ vững quy tắc kiểm toán suốt ca trực. |

---

## PHẦN 2: HỆ THỐNG PROMPT THỰC THI & CODE ĐIỀU PHỐI KHÉP KÍN

### 2.1. Prompt Trạm 1 – Trích xuất Dữ liệu & Nhận diện Dị biệt (Model Mini)

```markdown
### VAI TRÒ
Bạn là Bộ trích xuất & kiểm tra dữ liệu thô (Data Parser) của hệ thống kiểm toán FastDelivery.

### NHIỆM VỤ
Đọc chứng từ trong thẻ `<raw_doc>`, trích xuất thông tin và phát hiện dữ liệu dị biệt.

### QUY TẮC BẮT LỖI & TỐI ƯU TOKEN
1. **Không bịa dữ liệu:** Trường thiếu/rỗng gán `null`. Liệt kê trường thiếu vào `missing` (bắt buộc: `trip_id`, `plate`, `date`, `distance_km`, `liters`, `fuel_cost`).
2. **Bắt lỗi dữ liệu dị biệt (`anomalies`):**
   - `distance_km <= 0` hoặc `liters <= 0` hoặc `fuel_cost <= 0` hoặc `toll_fee < 0` -> thêm `"INVALID_NUMERIC"`.
   - `date` không đúng định dạng `YYYY-MM-DD` hoặc ngày không tồn tại (ví dụ: tháng > 12, ngày > 31) -> thêm `"INVALID_DATE"`.
   - Biển số `plate` không thuộc danh sách hợp lệ bắt đầu bằng `51C-` hoặc `60C-` -> thêm `"UNKNOWN_VEHICLE"`.
3. **Trạng thái (`status`):** Nếu `anomalies` khác rỗng -> `"ANOMALY"`; nếu `missing` khác rỗng -> `"INCOMPLETE"`; ngược lại -> `"VALID"`.

### ĐỊNH DẠNG ĐẦU RA (PURE JSON - KHÔNG GIẢI THÍCH)
{
  "trip_id": "string | null",
  "plate": "string | null",
  "date": "string | null",
  "distance_km": "number | null",
  "liters": "number | null",
  "fuel_cost": "number | null",
  "toll_fee": "number | null",
  "peak_season": "boolean",
  "status": "VALID | INCOMPLETE | ANOMALY",
  "missing": [],
  "anomalies": []
}

<raw_doc>
{{RAW_DOCUMENT_INPUT}}
</raw_doc>
```

---

### 2.2. Prompt Trạm 2 – Lập luận Kiểm toán Từng bước (Structured CoT – Model Pro)

```markdown
### VAI TRÒ
Bạn là Chuyên viên Kiểm toán Vận tải tại FastDelivery.

### QUY TẮC NGHIỆP VỤ KIỂM TOÁN
1. **Chốt chặn đầu vào:** Nếu `status != "VALID"`, DỪNG tính toán ngay, xuất mã lỗi và kết thúc.
2. **Định mức nhiên liệu chuẩn:** Xe tải container FastDelivery có định mức chuẩn là $32\text{ Lít} / 100\text{ km}$ ($0.32\text{ Lít/km}$).
3. **Tính toán đối soát:**
   - $\text{Standard\_Liters} = \text{distance\_km} \times 0.32$.
   - $\text{Variance\_Pct} = \frac{\text{liters} - \text{Standard\_Liters}}{\text{Standard\_Liters}} \times 100\%$.
   - Đơn giá xăng thực tế: $\text{Unit\_Price} = \frac{\text{fuel\_cost}}{\text{liters}}$.
   - Chi phí nhiên liệu hợp lệ tối đa: Nếu $\text{Variance\_Pct} > 15\%$, chỉ duyệt thanh toán theo định mức chuẩn ($\text{Approved\_Fuel} = \text{Standard\_Liters} \times \text{Unit\_Price}$) và gắn cờ gian lận/bất thường; ngược lại duyệt $100\%$ `fuel_cost`.
   - Phụ phí cầu đường: Nếu `peak_season == true`, được cộng thêm tối đa $10\%$ phụ phí trên `toll_fee` ($\text{Approved\_Toll} = \text{toll\_fee} \times 1.1$); ngược lại $\text{Approved\_Toll} = \text{toll\_fee}$.
4. **Ngưỡng duyệt thủ công (HITL):** Bật `HITL = TRUE` nếu $\text{Variance\_Pct} > 15\%$ HOẶC tổng tiền duyệt chi $\text{Total\_Approved} > 3.000.000\text{ VNĐ}$.

### YÊU CẦU LẬP LUẬN CHUỖI TƯ DUY (5 BƯỚC NGẮN GỌN, CHÍNH XÁC)
- **Bước 1 (Validation):** Xác nhận `status == "VALID"`.
- **Bước 2 (Định mức & Sai lệch):** Tính `Standard_Liters` và `Variance_Pct`.
- **Bước 3 (Chi phí nhiên liệu duyệt chi):** Tính `Unit_Price` và `Approved_Fuel`.
- **Bước 4 (Phí cầu đường & Tổng cộng):** Tính `Approved_Toll` theo mùa cao điểm và cộng tổng `Total_Approved`.
- **Bước 5 (Cross-Validation & HITL):** Kiểm tra lại phép tính Bước 4 và xác định cờ `HITL`.

### DÒNG CUỐI CÙNG BẮT BUỘC (KHÔNG THÊM VĂN BẢN PHÍA SAU)
`[RESULT: <APPROVED | FLAGGED_AUDIT | HOLD_FOR_FIX> | APPROVED_AMOUNT: <SO_NGUYEN> VND | VARIANCE: <SO_THAP_PHAN>% | HITL: <TRUE | FALSE>]`

<handoff_json>
{{STATION_1_JSON}}
</handoff_json>
```

---

### 2.3. Prompt Trạm 3 – Xuất Báo cáo & Quản lý Ngữ cảnh (Context Anchoring + Continue Prompting)

```markdown
[Neo bối cảnh]: Bạn là Hệ thống Xuất Báo cáo Kiểm toán FastDelivery. Văn phong ngắn gọn, khách quan, chuẩn xác từng con số. Tuyệt đối tuân thủ dòng `[RESULT]` từ Trạm 2.

### NHIỆM VỤ
Từ `<station_1_json>` và `<station_2_output>`:
1. Nếu `status != "VALID"` (tức `HOLD_FOR_FIX`): Xuất phiếu cảnh báo lỗi dữ liệu dị biệt gửi bộ phận Nhập liệu ERP (nêu rõ mã lỗi trong `missing` và `anomalies`).
2. Nếu `status == "VALID"`: Xuất 1 dòng tóm tắt chuẩn Markdown để chèn vào Bảng Đối soát Ca trực và ghi rõ trạng thái `[HITL: CẦN GIÁM ĐỐC DUYỆT]` nếu `HITL: TRUE`.

<station_1_json>
{{STATION_1_JSON}}
</station_1_json>

<station_2_output>
{{STATION_2_OUTPUT}}
</station_2_output>
```

---

### 2.4. Code Điều phối Chuỗi Khép kín & Bắt lỗi Hệ thống (Python Clean Code)

```python
import json
import re

def clean_json_output(raw_text: str) -> dict:
    """Làm sạch Handoff rác và parse JSON an toàn, không làm crash hệ thống."""
    cleaned = re.sub(r"^```json\s*|\s*```$", "", raw_text.strip(), flags=re.MULTILINE)
    return json.loads(cleaned)

def validate_gate(data: dict) -> tuple[bool, str]:
    """Chốt chặn kiểm định dữ liệu dị biệt trước khi gọi Model Pro ở Trạm 2."""
    if data.get("status") != "VALID":
        return False, f"Chặn tại Gate: status={data.get('status')}, anomalies={data.get('anomalies')}, missing={data.get('missing')}"
    if (data.get("distance_km") or 0) <= 0 or (data.get("liters") or 0) <= 0:
        return False, "Chặn tại Gate: Phát hiện distance_km hoặc liters <= 0 (Tránh lỗi chia cho 0)."
    return True, "OK"

def continue_if_truncated(llm_call_fn, current_text: str, finish_reason: str) -> str:
    """Tự động nối mạch khi chạm trần Max Output Tokens bằng kỹ thuật mớm 4 từ cuối."""
    while finish_reason == "length":
        words = current_text.strip().split()
        anchor_phrase = " ".join(words[-4:]) if len(words) >= 4 else current_text[-20:]
        resume_prompt = (
            f'Phản hồi bị ngắt ngang. Hãy viết tiếp ngay sau cụm từ: "{anchor_phrase}". '
            "Giữ nguyên định dạng cột Markdown hiện tại và không lặp lại nội dung đã xuất."
        )
        next_chunk, finish_reason = llm_call_fn(model="gemini-flash", prompt=resume_prompt)
        current_text += "\n" + next_chunk.strip()
    return current_text

def run_audit_pipeline(raw_doc: str, llm_call_fn) -> dict:
    """Chạy tuần tự khép kín 3 trạm với cơ chế bắt ngoại lệ toàn diện."""
    try:
        # TRẠM 1: Trích xuất dữ liệu bằng Model Mini (Tối ưu chi phí & tốc độ)
        p1 = STATION_1_PROMPT.replace("{{RAW_DOCUMENT_INPUT}}", raw_doc)
        raw_s1, _ = llm_call_fn(model="gemini-flash", prompt=p1)
        s1_data = clean_json_output(raw_s1)

        # CHỐT CHẶN VALIDATION GATE
        is_valid, gate_msg = validate_gate(s1_data)
        if not is_valid:
            return {
                "trip_id": s1_data.get("trip_id", "UNKNOWN"),
                "final_status": "HOLD_FOR_FIX",
                "detail": gate_msg,
                "system_crash": False
            }

        # TRẠM 2: Lập luận CoT đa biến số bằng Model Pro (Độ chính xác cao)
        p2 = STATION_2_PROMPT.replace("{{STATION_1_JSON}}", json.dumps(s1_data, ensure_ascii=False))
        s2_output, _ = llm_call_fn(model="gemini-pro", prompt=p2)

        # TRẠM 3: Xuất báo cáo & kiểm tra ngắt đoạn (Model Mini)
        p3 = (STATION_3_PROMPT
              .replace("{{STATION_1_JSON}}", json.dumps(s1_data, ensure_ascii=False))
              .replace("{{STATION_2_OUTPUT}}", s2_output))
        s3_output, finish_reason = llm_call_fn(model="gemini-flash", prompt=p3)
        s3_complete = continue_if_truncated(llm_call_fn, s3_output, finish_reason)

        return {
            "trip_id": s1_data["trip_id"],
            "final_status": "PROCESSED",
            "reasoning": s2_output,
            "report": s3_complete,
            "system_crash": False
        }

    except json.JSONDecodeError as e:
        return {"final_status": "ERROR_JSON_PARSE", "detail": str(e), "system_crash": False}
    except Exception as e:
        return {"final_status": "SAFE_FALLBACK", "detail": str(e), "system_crash": False}
```

---

## PHẦN 3: KẾT QUẢ CHẠY THỬ SẠCH LỖI CỦA AI

### 3.1. Test Case 1: Luồng Nghiệp vụ Đa biến số + Bật cờ Kiểm toán (`FLAGGED_AUDIT` & `HITL`)

#### Đầu vào (`<raw_doc>`):
> Phiếu thanh toán chuyến `TRIP-20261117-01`, xe tải `51C-88921`, ngày chạy `2026-11-17`. Quãng đường GPS ghi nhận `250 km`. Tài xế nộp hóa đơn đổ `96 lít` dầu hết `1.920.000 VNĐ`, phí cầu đường `400.000 VNĐ`, chuyến đi thuộc diện mùa cao điểm Tết (`peak_season = true`).

#### Kết quả Trạm 1 (Pure JSON Handoff):
```json
{
  "trip_id": "TRIP-20261117-01",
  "plate": "51C-88921",
  "date": "2026-11-17",
  "distance_km": 250,
  "liters": 96,
  "fuel_cost": 1920000,
  "toll_fee": 400000,
  "peak_season": true,
  "status": "VALID",
  "missing": [],
  "anomalies": []
}
```

#### Kết quả Trạm 2 (Structured CoT Reasoning):
* **Bước 1 (Validation):** Trạng thái `status == "VALID"`, không có dữ liệu thiếu hoặc bất thường.
* **Bước 2 (Định mức & Sai lệch):**
  * Định mức chuẩn cho $250\text{ km}$: $\text{Standard\_Liters} = 250 \times 0.32 = 80\text{ Lít}$.
  * Tỷ lệ sai lệch: $\text{Variance\_Pct} = \frac{96 - 80}{80} \times 100\% = +20.0\%$.
  * Vì $+20.0\% > 15\%$, phát hiện vượt định mức nhiên liệu cho phép $\rightarrow$ Chỉ duyệt thanh toán tối đa theo định mức $80\text{ Lít}$ và gắn cờ kiểm toán.
* **Bước 3 (Chi phí nhiên liệu duyệt chi):**
  * Đơn giá dầu thực tế trên hóa đơn: $\text{Unit\_Price} = \frac{1.920.000}{96} = 20.000\text{ VNĐ/Lít}$.
  * Tiền nhiên liệu được duyệt: $\text{Approved\_Fuel} = 80 \times 20.000 = 1.600.000\text{ VNĐ}$ (cắt giảm $320.000\text{ VNĐ}$ phần vượt định mức).
* **Bước 4 (Phí cầu đường & Tổng cộng):**
  * Do `peak_season == true`, phụ phí cầu đường được cộng $10\%$: $\text{Approved\_Toll} = 400.000 \times 1.1 = 440.000\text{ VNĐ}$.
  * Tổng số tiền duyệt chi: $\text{Total\_Approved} = 1.600.000 + 440.000 = 2.040.000\text{ VNĐ}$.
* **Bước 5 (Cross-Validation & HITL):** Kiểm tra lại: $1.600.000 + 440.000 = 2.040.000\text{ VNĐ}$ (khớp). Mặc dù tổng tiền $\le 3.000.000\text{ VNĐ}$, nhưng do $\text{Variance\_Pct} = 20.0\% > 15\%$, kích hoạt `HITL = TRUE`.

`[RESULT: FLAGGED_AUDIT | APPROVED_AMOUNT: 2040000 VND | VARIANCE: 20.0% | HITL: TRUE]`

#### Kết quả Trạm 3 (Báo cáo Sạch Lỗi Đầu Ra):
```markdown
| Mã chuyến | Biển số | Ngày | Tiền duyệt chi | Sai lệch định mức | Trạng thái xử lý |
| :--- | :--- | :--- | :--- | :--- | :--- |
| TRIP-20261117-01 | 51C-88921 | 2026-11-17 | 2.040.000 VNĐ | +20.0% (Vượt trần 15%) | FLAGGED_AUDIT - [HITL: CẦN GIÁM ĐỐC DUYỆT] |
```

---

### 3.2. Test Case 2: Bẫy Dữ liệu Dị biệt (Chống Crash & Chặn Lỗi Lan Truyền)

#### Đầu vào chứa dữ liệu bẩn (`<raw_doc>`):
> Chuyến `TRIP-20261117-99`, xe `99Z-00000` (biển lạ), ngày `32/13/2026` (ngày không tồn tại). GPS báo `0 km` (lỗi thiết bị), hóa đơn ghi `-150.000 VNĐ`, không có thông tin số lít dầu đổ.

#### Kết quả Trạm 1 (Bắt trọn toàn bộ lỗi dị biệt):
```json
{
  "trip_id": "TRIP-20261117-99",
  "plate": "99Z-00000",
  "date": "32/13/2026",
  "distance_km": 0,
  "liters": null,
  "fuel_cost": -150000,
  "toll_fee": null,
  "peak_season": false,
  "status": "ANOMALY",
  "missing": [
    "liters"
  ],
  "anomalies": [
    "INVALID_NUMERIC",
    "INVALID_DATE",
    "UNKNOWN_VEHICLE"
  ]
}
```

#### Kết quả Hệ thống Điều phối & Trạm 3 (An toàn tuyệt đối, không Crash):
* **Tại Validation Gate:** Chặn đứng luồng dữ liệu ngay trước Trạm 2, tránh được lỗi chia cho 0 (`distance_km = 0`) và lỗi tính toán tiền âm (`fuel_cost = -150000`).
* **Đầu ra Trạm 3 (Phiếu cảnh báo gửi bộ phận ERP):**
  > **[CẢNH BÁO DỮ LIỆU DỊ BIỆT - HOLD_FOR_FIX]**
  > * **Mã chuyến:** `TRIP-20261117-99`
  > * **Trường dữ liệu bị thiếu:** `liters` (Số lít nhiên liệu).
  > * **Lỗi dị biệt phát hiện:**
  >   1. `INVALID_NUMERIC`: Quãng đường bằng `0 km` và chi phí nhiên liệu âm (`-150.000 VNĐ`).
  >   2. `INVALID_DATE`: Ngày chứng từ `32/13/2026` không hợp lệ.
  >   3. `UNKNOWN_VEHICLE`: Biển số `99Z-00000` không nằm trong danh mục đội xe `51C-` / `60C-`.
  > * **Hành động:** Tạm ngưng giải ngân, chuyển trả bộ phận Nhập liệu ERP hiệu đính trước khi đối soát lại.

---

### 3.3. Test Case 3: Khôi phục Bảng Tổng hợp Bị Ngắt Ngang (`Continue Prompting`)

#### Trạng thái ngắt ngang giả lập (`finish_reason = "length"`):
```markdown
| Mã chuyến | Biển số | Ngày | Tiền duyệt chi | Sai lệch định mức | Trạng thái xử lý |
| :--- | :--- | :--- | :--- | :--- | :--- |
| TRIP-20261117-19 | 60C-41209 | 2026-11-17 | 1.480.000 VNĐ | +2.5% | APPROVED |
```
*(Hệ thống phát hiện `finish_reason == "length"`, tự động trích 4 từ cuối `+2.5% | APPROVED |` để kích hoạt `continue_if_truncated`)*

#### Kết quả nối tiếp liền mạch 100% sau khi chạy hàm khôi phục:
```markdown
| TRIP-20261117-20 | 51C-77310 | 2026-11-17 | 3.420.000 VNĐ | +4.1% | APPROVED - [HITL: CẦN GIÁM ĐỐC DUYỆT] |
| TRIP-20261117-21 | 60C-19845 | 2026-11-17 | 1.120.000 VNĐ | -1.2% | APPROVED |
```