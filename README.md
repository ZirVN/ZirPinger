# ZirPinger

**Server-side Ping Equalizer cho Minecraft**  
Tăng ping thật sự của người chơi thông qua Netty pipeline — tạo môi trường PvP công bằng giữa những người có ping chênh lệch nhau.

> **Version:** 2.0.0  
> **Author:** ZirVN 
> **Server:** Paper / Spigot **1.21 – 1.21.8**  
> **Java:** 17+  
> **Depends:** [PacketEvents](https://modrinth.com/plugin/packetevents) 2.11.2+

---

## Cài đặt

1. Tải **PacketEvents 2.11.2+** và bỏ vào `plugins/`
2. Tải **ZirPinger.jar** và bỏ vào `plugins/`
3. Khởi động server — tự tạo `config.yml` và `messages.yml`

> ⚠️ ZirPinger **bắt buộc** có PacketEvents mới chạy được. Thiếu PacketEvents → server từ chối load plugin.

---

## Cách hoạt động

ZirPinger chèn một handler (`zirpinger_delay`) vào **Netty ChannelPipeline** của từng kết nối player, đặt ngay trước `encoder`:

```
[Server logic] → [compressor] → [encryptor] → [zirpinger_delay] → [encoder] → [socket]
```

Mọi packet server gửi xuống client đều bị **giữ lại** trong `delayMs` milliseconds trước khi thực sự gửi đi. Kết quả là client nhận thông tin muộn hơn → phản hồi muộn hơn → server đo RTT cao hơn thật sự. Đây là delay thật, không phải visual.

Channel được lấy qua **PacketEvents API** — không dùng reflection NMS — nên hoạt động ổn định trên mọi version 1.21.x mà không bị break khi server update.

**Giới hạn cần biết:**
- Chỉ delay chiều **server → client** (outbound). Chiều client → server không bị chặn.
- **Không thể giảm ping** — chỉ có thể tăng. Người có ping 150ms không thể kéo xuống 100ms.
- Ping hiển thị trong tab-list của chính người chơi đó có thể không đổi (ICMP riêng), nhưng keepalive RTT mà server đo được tăng thật sự.

---

## Lệnh

### `/zp` — Lệnh chính
Alias: `/zirpinger`  
Permission: `zirpinger.admin`

| Lệnh | Mô tả |
|---|---|
| `/zp help` | Hiện menu trợ giúp |
| `/zp set <player> add <ms>` | Thêm cố định `<ms>` vào ping gốc của player |
| `/zp set <player> total <ms>` | Tự tính và áp delay để đạt tổng `<ms>` |
| `/zp set <player> off` | Tắt delay cho player |
| `/zp group <player> <group>` | Áp dụng preset group từ config |
| `/zp status [player]` | Xem ping gốc / delay đang áp / tổng ping |
| `/zp list` | Danh sách tất cả player đang có delay |
| `/zp groups` | Danh sách preset groups đã cấu hình |
| `/zp reset` | Xóa delay của toàn bộ player |
| `/zp reload` | Reload `config.yml` và `messages.yml` |

---

### `/zpset` — Shorthand nhanh
Permission: `zirpinger.admin`

```
/zpset <player> add <ms>
/zpset <player> total <ms>
/zpset <player> off
```

Tương đương với `/zp set ...` — dùng khi cần gõ nhanh.

---

### `/zpstatus [player]`
Permission: `zirpinger.use` (admin xem được người khác, player chỉ xem được bản thân)

Hiển thị:
- Ping gốc đang đo được
- Delay đang áp dụng
- Tổng ping ước tính
- Chế độ hiện tại (OFF / ADD / TOTAL)

---

### `/zpme`
Permission: `zirpinger.use`

Player tự xem trạng thái ping của mình. Hiển thị thêm ping Netty thực đo bởi server.

---

### `/zpeq` — Global Equalizer
Alias: `/equalizer`, `/zpequalizer`  
Permission: `zirpinger.admin`

Tự động cân bằng ping toàn server về một mức target. Áp dụng cho tất cả player hiện tại và mỗi người vào sau.

| Lệnh | Mô tả |
|---|---|
| `/zpeq on <ms>` | Bật equalizer, đặt ping target = `<ms>` |
| `/zpeq off` | Tắt equalizer, xóa tất cả delay đã áp |
| `/zpeq status` | Bảng trạng thái + danh sách từng player online |
| `/zpeq target <ms>` | Đổi ping target mà không cần tắt/bật lại |
| `/zpeq tolerance <ms>` | Đặt ngưỡng dung sai (mặc định `10`) |
| `/zpeq highping on <ms>` | Cộng thêm `<ms>` cho người có ping cao hơn target |
| `/zpeq highping off` | Không can thiệp người có ping cao hơn target |
| `/zpeq check <player>` | Xem chi tiết delay equalizer của 1 player |
| `/zpeq apply <player>` | Áp thủ công equalizer cho 1 player |

**Ví dụ thực tế — `/zpeq on 100`:**

| Player | Ping gốc | Delay thêm | Kết quả |
|---|---|---|---|
| Anh A | 25ms | +75ms | ~100ms |
| Anh B | 45ms | +55ms | ~100ms |
| Anh C | 65ms | +35ms | ~100ms |
| Anh D | 120ms | 0ms | 120ms (không thể giảm) |

**Tolerance** (`/zpeq tolerance 10`): player có ping trong khoảng `90–110ms` sẽ không bị thêm delay — tránh dao động nhỏ gây inject/remove handler liên tục.

**Highping** (`/zpeq highping on 20`): player có ping trên target (Anh D, 120ms) sẽ bị cộng thêm 20ms nữa. Dùng khi muốn mọi người đều có delay nhất định thay vì để Anh D hoàn toàn tự nhiên.

---

## Permissions

| Permission | Mô tả | Mặc định |
|---|---|---|
| `zirpinger.admin` | Dùng toàn bộ lệnh set / eq / reset / reload | OP |
| `zirpinger.use` | Xem trạng thái ping của bản thân (`/zpme`, `/zpstatus`) | Tất cả |
| `zirpinger.bypass` | Miễn trừ khỏi mọi delay — kể cả Global Equalizer | Không ai |
| `zirpinger.notify` | Nhận thông báo mỗi khi admin thay đổi delay của ai đó | OP |

---

## Cấu hình — `config.yml`

```yaml
settings:
  min-delay: 0          # Delay tối thiểu được phép đặt (ms)
  max-delay: 500        # Delay tối đa được phép đặt (ms)
  announce-changes: true  # Broadcast ra chat khi bật/tắt delay
  log-changes: true       # Ghi log console khi delay thay đổi
  calibration-interval: 40  # Tần suất hiệu chỉnh delay (ticks, 40 = 2 giây)

quantization-step: 2    # Delay được làm tròn về bội số của 2ms
debug: false            # Bật để in log calibration chi tiết
```

### Groups (Preset)

Định nghĩa sẵn, áp dụng bằng `/zp group <player> <tên>`:

```yaml
groups:
  high-ping:
    mode: total       # "add" = cộng thêm, "total" = tổng mục tiêu
    amount: 150
    description: "Simulate 150ms ping"
  medium-ping:
    mode: total
    amount: 100
    description: "Simulate 100ms ping"
  low-ping:
    mode: add
    amount: 30
    description: "Add 30ms extra"
  pvp-equal:
    mode: total
    amount: 120
    description: "PvP equalization - 120ms"
```

Thêm group mới tùy ý, `/zp reload` để áp dụng không cần restart.

### Global Equalizer

```yaml
global-equalizer:
  enabled: false          # true = bật ngay khi server khởi động
  target-ping: 100        # Mục tiêu ping (ms)
  tolerance: 10           # ±10ms — trong vùng này không thêm delay
  equalize-high-ping: false   # Có thêm delay cho người ping cao hơn target?
  high-ping-add-ms: 0         # Nếu có, thêm bao nhiêu ms
```

### Player Overrides

Tự động được plugin lưu mỗi khi dùng `/zp set`:

```yaml
player-overrides:
  550e8400-e29b-41d4-a716-446655440000:
    mode: TOTAL
    amount: 100
```

Xóa entry này để xóa override của người chơi đó khi offline.

---

## Cấu hình — `messages.yml`

Tất cả text hiển thị đều nằm trong `messages.yml`. Hỗ trợ màu Minecraft (`&a`, `&e`, `&c`...) và các placeholder:

| Placeholder | Giá trị |
|---|---|
| `{player}` | Tên người chơi mục tiêu |
| `{delay}` | Delay đang áp dụng (ms) |
| `{mode}` | Chế độ hiện tại (ADD / TOTAL / OFF) |
| `{total}` | Tổng ping ước tính |
| `{base}` | Ping gốc đo được |
| `{group}` | Tên group |
| `{admin}` | Tên admin thực hiện lệnh |
| `{input}` | Giá trị người dùng nhập (dùng trong thông báo lỗi) |
| `{min}` / `{max}` | Giới hạn delay trong config |
| `{value}` | Giá trị ms broadcast |

Chỉnh xong, dùng `/zp reload` để áp dụng ngay mà không cần restart.

---

## Hai chế độ delay — sự khác biệt

**ADD mode** — cộng cố định:
```
delay áp dụng = addAmount  (luôn cố định, bất kể ping gốc thay đổi)
```
Ping gốc 25ms, add 75ms → tổng ~100ms.  
Ping gốc sau đó tăng lên 40ms, add vẫn là 75ms → tổng ~115ms.  
→ Dùng khi muốn thêm một lượng cố định, không quan tâm tổng là bao nhiêu.

**TOTAL mode** — nhắm vào tổng:
```
delay áp dụng = max(0, totalTarget - basePing)
```
Ping gốc 25ms, target 100ms → delay = 75ms → tổng ~100ms.  
Ping gốc sau đó tăng lên 40ms → delay tự điều chỉnh xuống 60ms → tổng vẫn ~100ms.  
→ Dùng khi muốn cố định tổng ping bất kể mạng người chơi biến động.

Delay trong TOTAL mode được cập nhật tự động mỗi `calibration-interval` ticks bằng **EWMA smoothing** (tránh nhảy đột ngột).

---

