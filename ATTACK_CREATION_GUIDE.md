# Hướng Dẫn Tạo Chiêu Cho Sans - UTEngine

Tài liệu này cung cấp hướng dẫn chi tiết cách tạo chiêu (attack) mới cho Sans trong UTEngine.

---

## Mục Lục
1. [Kiến Thức Nền Tảng](#kiến-thức-nền-tảng)
2. [Cấu Trúc File Attack](#cấu-trúc-file-attack)
3. [Các Phương Thức AttackManager](#các-phương-thức-attackmanager)
4. [Hướng Dẫn Từng Bước](#hướng-dẫn-từng-bước)
5. [Ví Dụ Mẫu](#ví-dụ-mẫu)
6. [Đăng Ký Chiêu Vào Battle](#đăng-ký-chiêu-vào-battle)
7. [Tips & Debugging](#tips--debugging)

---

## Kiến Thức Nền Tảng

### Attack Lifecycle (Vòng Đời Chiêu)

Mỗi chiêu trải qua các giai đoạn:

1. **pre_attack()** — Chuẩn bị (setup HUD, hiển thị đối thoại, set battle box)
2. **start_attack()** — Bắt đầu (spawn đạn/chiêu, bật `attack_started = true`)
3. **_process(delta)** — Xử lý frame (đếm frame, gọi `end_attack()` khi xong)
4. **end_attack()** — Kết thúc (hiển thị đối thoại, phát hành control, dọn dẹp)

### Cấu Trúc Class Attack

```gdscript
extends Attack  # Kế thừa từ Attack (xem scripts/battle/attacks/attack_base.gd)

var a_vars : vars = vars  # Reference đến autoload vars
var attack_started := false  # Từ Attack base class
var current_frames := 0.0  # Từ Attack base class
var frames := 360.0  # Độ dài attack (frames), sửa trong _init()

signal attack_finished  # Từ Attack base class
```

---

## Cấu Trúc File Attack

Mỗi file attack nằm ở `scripts/battle/attacks/` và extends `Attack`.

**File mẫu: `my_custom_attack.gd`**

```gdscript
extends Attack

var a_vars : vars = vars

func _init():
    frames = 600  # Độ dài attack = 600 frames = ~10 giây (60 FPS)

func pre_attack():
    # Chuẩn bị HUD
    a_vars.hud_manager.mode = -1
    a_vars.player_heart.visible = false
    
    # Hiển thị đối thoại (nếu cần)
    a_vars.main_writer.set_font("sans", 24)
    a_vars.main_writer.writer_text = "(face:sans/normal)(sound:sans)Your text here(pc)"
    await a_vars.main_writer.done
    
    # Setup battle box và heart
    a_vars.player_heart.heart_mode = PlayerHeart.e_heart_mode.red  # hoặc blue
    a_vars.battle_box.set_box_size([244,250,399,390], 300)  # [min_x, min_y, max_x, max_y], duration_ms
    await get_tree().process_frame
    a_vars.player_heart.visible = true
    a_vars.player_heart.global_position = Vector2(321, 324)

func start_attack():
    a_vars.player_heart.input_enabled = true
    attack_started = true
    
    # Spawn pattern của chiêu
    # ... (xem phần Các Phương Thức AttackManager)

func end_attack():
    # Hiển thị đối thoại sau khi chiêu (nếu cần)
    a_vars.hud_manager.mode = -1
    a_vars.player_heart.visible = false
    a_vars.main_writer.set_font("sans", 24)
    a_vars.main_writer.writer_text = "(face:sans/normal)(sound:sans)End text(pc)"
    await a_vars.main_writer.done
    
    # Reset HUD và kết thúc
    a_vars.hud_manager.reset()
    attack_finished.emit()
    queue_free()

func _process(delta):
    if(attack_started):
        current_frames += delta * 60
        if(current_frames > frames):
            end_attack()
```

---

## Các Phương Thức AttackManager

Tất cả các phương thức dưới đây gọi qua `a_vars.attack_manager.method_name(...)`.

### Xương (Bones)

#### 1. Xương đơn lẻ
```gdscript
a_vars.attack_manager.bone(
    type,           # Bullet.e_type: 0=none, 1=blue, 2=fake_blue, 3=orange, 4=unhittable
    position,       # Vector2 (x, y) vị trí spawn
    x,              # Tốc độ X (ngang)
    y,              # Tốc độ Y (dọc)
    speed,          # Độ lớn tốc độ
    offset_top,     # Offset trên
    offset_bottom,  # Offset dưới
    rotation_speed, # Tốc độ quay
    masked,         # true = hiển thị phía sau, false = phía trước (default: true)
    duration        # Thời gian tồn tại (-1 = vô hạn)
)
```

**Ví dụ:**
```gdscript
a_vars.attack_manager.bone(0, Vector2(150, 254), 2, 0, 120, 0, 50, 0, true)
```

#### 2. Vòng xương quay
```gdscript
a_vars.attack_manager.bone_circle(
    type,           # Bullet.e_type
    position,       # Tâm vòng tròn
    bone_count,     # Số lượng xương
    radius,         # Bán kính vòng
    rotation_speed, # Tốc độ quay
    masked,         # (default: true)
    duration        # (default: -1)
)
```

**Ví dụ:**
```gdscript
a_vars.attack_manager.bone_circle(1, Vector2(320, 180), 12, 100, 120, true)
# 12 xương xanh xoay quanh (320, 180) với bán kính 100
```

#### 3. Xương "đâm lên" (Bone Stab)
```gdscript
a_vars.attack_manager.bone_stab(
    type,           # Bullet.e_type
    position,       # Vị trí
    length,         # Chiều dài
    height,         # Chiều cao
    wait_time,      # Thời gian chờ (frame)
    up_time,        # Thời gian đâm lên (frame)
    bone_rotation,  # Góc quay
    masked          # (default: true)
)
```

**Ví dụ:**
```gdscript
a_vars.attack_manager.bone_stab(0, Vector2(244, 260), 140, 50, 10, 24, 0, true)
```

#### 4. Xương rơi (Bone Gravity)
```gdscript
a_vars.attack_manager.bone_gravity(
    type,           # Bullet.e_type
    position,       # Vị trí
    bone_count,     # Số lượng
    offset_bottom,  # Offset
    masked,         # (default: false)
    duration        # (default: -1)
)
```

---

### Gaster Blaster

```gdscript
a_vars.attack_manager.gaster_blaster(
    type,           # Bullet.e_type
    start_position, # Vector2 (vị trí mở)
    end_position,   # Vector2 (vị trí bắn)
    end_rotation,   # Góc bắn (độ)
    scale,          # Vector2 (tỷ lệ kích cỡ)
    wait_time,      # Thời gian chờ trước bắn (s)
    blast_time,     # Thời gian bắn (s)
    masked          # (default: false)
)
```

**Ví dụ:**
```gdscript
a_vars.attack_manager.gaster_blaster(0, Vector2(-100,-100), Vector2(150,120), -45, Vector2(1,1), 0.2, 0.5, false)
```

---

### Vector Slash

```gdscript
a_vars.attack_manager.vector_slash(
    type,                # Bullet.e_type
    position,            # Vector2
    wait_time,           # Thời gian chờ (s)
    starting_rotation,   # Góc bắt đầu
    rotation_speed,      # Tốc độ quay
    stop_rotation_after, # Dừng quay sau khi hit
    masked               # (default: false)
)
```

---

### Platform

```gdscript
a_vars.attack_manager.platform(
    platform_type,  # BPlatform.e_platform_type (kiểu nền)
    position,       # Vector2
    x,              # Tốc độ X
    y,              # Tốc độ Y
    speed,          # Độ lớn tốc độ
    masked,         # (default: false)
    duration        # (default: -1)
)
```

---

### Hiệu Ứng Đặc Biệt

#### Quả tim bị ném
```gdscript
a_vars.attack_manager.throw(
    direction,  # Góc ném (độ, 0=phải, 90=trên, 180=trái, 270=dưới)
    fall_speed  # Tốc độ rơi (default: 750)
)
```

#### Cảnh báo (Warning)
```gdscript
a_vars.attack_manager.warning(
    position,  # Vector2
    size,      # Vector2 (chiều rộng, chiều cao)
    duration,  # Thời gian hiển thị (s)
    masked     # (default: true)
)
```

#### Màn hình đen
```gdscript
a_vars.attack_manager.black_screen(time)  # time = độ dài (s)
```

#### Xóa đạn
```gdscript
a_vars.attack_manager.delete_bullets.emit()
```

---

## Hướng Dẫn Từng Bước

### Bước 1: Tạo File Script Mới

1. Mở VS Code (hoặc editor yêu thích)
2. Tạo file mới: `scripts/battle/attacks/my_attack_name.gd`
3. Paste template từ phần [Cấu Trúc File Attack](#cấu-trúc-file-attack) ở trên

### Bước 2: Thiết Kế Pattern Chiêu

Quyết định pattern:
- Loại đạn nào: xương? gaster? platform?
- Bao nhiêu sóng (wave)?
- Độ khó: 1-10?
- Tương tác: có throw heart không?

**Ví dụ pattern:**
- Wave 1: Xương quay (8 xương)
- Wave 2: Gaster blasters từ 2 bên (5 cái)
- Wave 3: Xương đâm (3 cái)
- Thời gian: 10 giây tổng cộng

### Bước 3: Viết Code Chiêu

```gdscript
func start_attack():
    a_vars.player_heart.input_enabled = true
    attack_started = true
    
    # Wave 1: Xương quay
    a_vars.attack_manager.bone_circle(1, Vector2(320, 180), 8, 100, 100, true)
    await get_tree().create_timer(2.0).timeout
    
    # Wave 2: Gaster blasters
    for i in range(5):
        a_vars.attack_manager.gaster_blaster(0, Vector2(-100,-100), Vector2(150 + i*40, 150), -i*15, Vector2(1,1), 0.1*i, 0.4, false)
        await get_tree().create_timer(0.15).timeout
    
    await get_tree().create_timer(1.0).timeout
    
    # Wave 3: Xương đâm
    for x in [220, 280, 340]:
        a_vars.attack_manager.bone_stab(0, Vector2(x, 260), 140, 50, 10, 20, 0, true)
        await get_tree().create_timer(0.2).timeout
```

### Bước 4: Thêm Đối Thoại (Nếu Cần)

**Trước chiêu (pre_attack):**
```gdscript
a_vars.main_writer.writer_text = "(face:sans/normal)(sound:sans)Đây là chiêu của tôi!(pc)"
await a_vars.main_writer.done
```

**Giữa chiêu (start_attack):**
```gdscript
# ... some waves ...
await get_tree().create_timer(3.0).timeout
a_vars.main_writer.writer_text = "(face:sans/normal)(sound:sans)Còn tiếp nữa...(pc)"
await a_vars.main_writer.done
# ... more waves ...
```

**Sau chiêu (end_attack):**
```gdscript
a_vars.main_writer.writer_text = "(face:sans/normal)(sound:sans)Xong rồi!(pc)"
await a_vars.main_writer.done
```

### Bước 5: Đăng Ký Chiêu (Xem phần Đăng Ký Chiêu Vào Battle)

---

## Ví Dụ Mẫu

### Ví Dụ 1: Chiêu Đơn Giản (Xương Quay)

```gdscript
extends Attack

var a_vars : vars = vars

func _init():
    frames = 300

func pre_attack():
    a_vars.hud_manager.mode = -1
    a_vars.player_heart.visible = false
    
    a_vars.player_heart.heart_mode = PlayerHeart.e_heart_mode.red
    a_vars.battle_box.set_box_size([244,250,399,390], 200)
    await get_tree().process_frame
    a_vars.player_heart.visible = true
    a_vars.player_heart.global_position = Vector2(321, 324)

func start_attack():
    a_vars.player_heart.input_enabled = true
    attack_started = true
    
    a_vars.attack_manager.bone_circle(1, Vector2(320, 200), 10, 80, 150, true)
    await get_tree().create_timer(3.0).timeout
    a_vars.attack_manager.delete_bullets.emit()

func end_attack():
    a_vars.hud_manager.reset()
    attack_finished.emit()
    queue_free()

func _process(delta):
    if(attack_started):
        current_frames += delta * 60
        if(current_frames > frames):
            end_attack()
```

### Ví Dụ 2: Chiêu Phức Tạp (Multi-Wave)

```gdscript
extends Attack

var a_vars : vars = vars

func _init():
    frames = 600

func pre_attack():
    a_vars.hud_manager.mode = -1
    a_vars.player_heart.visible = false
    a_vars.main_writer.set_font("sans", 24)
    a_vars.main_writer.writer_text = "(face:sans/normal)(sound:sans)Đến lượt tôi rồi.(pc)"
    await a_vars.main_writer.done
    
    a_vars.player_heart.heart_mode = PlayerHeart.e_heart_mode.blue
    a_vars.battle_box.set_box_size([244,250,399,390], 300)
    await get_tree().process_frame
    a_vars.player_heart.visible = true
    a_vars.player_heart.global_position = Vector2(321, 324)

func start_attack():
    a_vars.player_heart.input_enabled = true
    attack_started = true
    
    # Wave 1: Xương quay
    a_vars.attack_manager.bone_circle(0, Vector2(320, 180), 10, 100, 120, true)
    await get_tree().create_timer(1.5).timeout
    
    # Wave 2: Gaster blasters
    for i in range(4):
        a_vars.attack_manager.gaster_blaster(0, Vector2(-100,-100), Vector2(150, 120 + i*30), -i*20, Vector2(1,1), 0.1*i, 0.5, false)
        await get_tree().create_timer(0.15).timeout
    
    await get_tree().create_timer(1.0).timeout
    
    # Wave 3: Xương đâm
    for x in [240, 280, 320, 360]:
        a_vars.attack_manager.bone_stab(0, Vector2(x, 265), 130, 45, 15, 25, 0, true)
        await get_tree().create_timer(0.2).timeout

func end_attack():
    a_vars.hud_manager.mode = -1
    a_vars.player_heart.visible = false
    a_vars.main_writer.writer_text = "(face:sans/normal)(sound:sans)Không tệ lắm.(pc)"
    await a_vars.main_writer.done
    
    a_vars.hud_manager.reset()
    attack_finished.emit()
    queue_free()

func _process(delta):
    if(attack_started):
        current_frames += delta * 60
        if(current_frames > frames):
            end_attack()
```

---

## Đăng Ký Chiêu Vào Battle

### Cách 1: Thêm Vào Battle Example (Đơn Giản)

1. Mở scene: `res://scenes/battles/example_battles/battle_example.tscn`
2. Tìm SubResource `GDScript_c0lm0` (AttackManager script)
3. Sửa dòng `attacks = [...]`:

```gdscript
func _init():
    attacks = [preload("res://scripts/battle/attacks/sans_new_attack.gd")]
    attacks.append(preload("res://scripts/battle/attacks/my_attack_name.gd"))
```

### Cách 2: Gọi Chiêu Custom Lúc Runtime

Trong code battle (hoặc scene script):

```gdscript
var my_attack_script = preload("res://scripts/battle/attacks/my_attack_name.gd")
vars.attack_manager.pre_custom_attack(my_attack_script)
```

---

## Tips & Debugging

### Tips 1: Điều Chỉnh Thông Số

- **Tốc độ xương:** `rotation_speed` hoặc `speed`
- **Số lượng xương:** `bone_count` trong `bone_circle()`
- **Bán kính vòng:** `radius` trong `bone_circle()`
- **Độ dài attack:** `frames` trong `_init()`
- **Delay giữa waves:** `await get_tree().create_timer(seconds).timeout`

### Tips 2: Kiểm Tra Kiểu Đạn

Bullet types từ `scripts/battle/bullet.gd`:
- `0` = `none` (trắng, bình thường)
- `1` = `blue` (xanh, phải di chuyển để tránh)
- `2` = `fake_blue` (xanh, phải bấm phím)
- `3` = `orange` (cam, phải không di chuyển để tránh)
- `4` = `unhittable` (không thể hit)

### Tips 3: Debug & Testing

1. **Mở Godot:**
   - Load project
   - Chạy scene `res://scenes/battles/example_battles/battle_example.tscn`
   - Nhấn Play Scene

2. **Kiểm Tra Output:**
   - Mở Output tab
   - Tìm lỗi (lỗi enum, gọi function không tồn tại, v.v)

3. **Chỉnh Chiêu:**
   - Thay đổi thông số trong script
   - Save file
   - Godot tự reload (hot reload)
   - Chạy scene lại

### Tips 4: Lỗi Thường Gặp

| Lỗi | Nguyên Nhân | Cách Sửa |
|-----|-----------|---------|
| "Function not found" | Gọi phương thức không tồn tại | Kiểm tra tên phương thức trong `AttackManager` |
| "Cannot find member" | Enum value sai | Dùng `0-4` thay vì `Bullet.e_type.xxx` |
| Chiêu không hiển thị | `attack_started` chưa bật hoặc `pre_attack()` chưa chạy | Đảm bảo gọi `start_attack()` sau `pre_attack()` |
| Đạn vẫn còn sau khi chiêu xong | Quên gọi `delete_bullets.emit()` | Thêm vào cuối `start_attack()` hoặc `end_attack()` |
| Heart bị lạc | Vị trí sai | Kiểm tra `a_vars.player_heart.global_position` |

### Tips 5: Export Variables (Nâng Cao)

Để dễ chỉnh thông số mà không cần sửa code:

```gdscript
@export var bone_count := 10
@export var rotation_speed := 120.0
@export var attack_duration := 600

func _init():
    frames = attack_duration

func start_attack():
    a_vars.player_heart.input_enabled = true
    attack_started = true
    a_vars.attack_manager.bone_circle(0, Vector2(320, 180), bone_count, 100, rotation_speed, true)
```

(Note: Export chỉ hoạt động khi script được dùng làm scene, không áp dụng cho attack scripts)

---

## Tài Liệu Liên Quan

- **Attack Base Class:** `scripts/battle/attacks/attack_base.gd`
- **Attack Manager:** `scripts/battle/attack_manager.gd`
- **Bullet Class:** `scripts/battle/bullet.gd`
- **Player Heart:** `scripts/battle/player_heart.gd`
- **Writer (Hội Thoại):** `scripts/global/writer.gd`
- **Battle Room:** `scripts/battle/battle_room.gd`

---

## Ghi Chú Cuối

- Luôn gọi `end_attack()` hoặc `attack_finished.emit()` để kết thúc chiêu.
- Dùng `await` để đồng bộ events (không dùng callback nếu có thể).
- Thử nghiệm thường xuyên trong Godot editor.
- Lưu file script trước khi chạy scene.

Chúc bạn tạo chiêu vui vẻ! 🎮

