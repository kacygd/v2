SANs Animation — Hướng dẫn thay đổi animation cho Sans (UTEngine)

Mục đích
- Tập hợp cách thay đổi, thêm và điều khiển animation cho Sans trong codebase này.
- Bao gồm: vị trí tài nguyên, chỉnh code, ví dụ thay đổi frame và dùng tween.

1) File & node liên quan
- Scene chính: `res://scenes/battles/example_battles/battle_example.tscn`
  - Node `enemies/example_enemy` có các child: `legs`, `torso`, `head` (kiểu AnimatedSprite2D)
  - Speech bubble: `enemies/example_enemy/speech_bubble` + `speech_writer` (RichTextLabel)
- Script của enemy (subresource trong TSCN): `GDScript_ohi4s` (extends Enemy). Đây là nơi logic hiển thị frame/animation hiện tại.
- Sprite frames: `assets/sprites/battle/enemies/sans_undertale/*` (textures + sprite frames -> dùng trong AnimatedSprite2D)

2) Làm quen với AnimatedSprite2D API trong code
- Đổi animation / frame:
  - ` $head.play("default") ` hoặc ` $head.play("flash") `
  - ` $head.frame = 2 ` để set frame cụ thể
  - ` $head.set_sprite_frames(load("res://path/to/spriteframes.tres")) ` để đổi resource
- Kiểm tra/đổi từ script (ví dụ trong subresource script):
  - `enemy.get_node("head").frame = 3`
  - `enemy.get_node("torso").frame = 1`

3) Thay đổi animation ở runtime (ví dụ blink, biểu cảm)
- Blink đơn giản (thay frame rồi chờ):
```
var head = $head
head.frame = 1 # mở mắt
await get_tree().create_timer(0.1).timeout
head.frame = 0 # nhắm mắt
```
- Dùng AnimatedSprite2D.play():
  - Trong `SpriteFrames` tạo animation `blink` gồm các frame mong muốn, sau đó gọi `head.play("blink")`.

4) Thêm animation mới (bước cơ bản)
- Mở `assets/sprites/battle/enemies/sans_undertale/` trong Godot
- Tạo một `SpriteFrames` resource mới hoặc chỉnh `SpriteFrames` hiện có:
  - Thêm animation name (ví dụ `angry`), kéo thả textures vào frames
- Trong scene (hoặc code) set `head.sprite_frames = load("res://.../my_sans_frames.tres")`
- Gọi `head.play("angry")` để chạy animation

5) Dùng Tween để thay đổi vị trí/offset/head movement mượt
- Ví dụ muốn head lắc nhẹ:
```
var t = create_tween()
t.tween_property($head, "position:x", $head.position.x + 4, 0.08).set_trans(Tween.TRANS_SINE).set_ease(Tween.EASE_IN_OUT)
```
- Hoặc để làm torso bobbing (giữ nguyên code pattern trong `GDScript_ohi4s`):
  - Trong `_process(delta)` có thể cập nhật `torso_offset` bằng `sin()` để tạo chuyển động liên tục.

6) Ví dụ: thêm blink ngẫu nhiên sau mỗi 3-6 giây
```gdscript
func _ready():
    randomize()
    start_blink_loop()

func start_blink_loop():
    while true:
        var wait = randf_range(3.0, 6.0)
        await get_tree().create_timer(wait).timeout
        $head.play("blink")
        await get_tree().create_timer(0.2).timeout
        $head.play("default")
```

7) Thay đổi mặt/sprite khi hội thoại
- `speech_bubble` dùng `speech_writer` để hiển thị text, nhưng mặt `head` có thể đổi frame đồng thời:
```
enemy.speech_bubble.visible = true
enemy.get_node("head").frame = 3 # biểu cảm
enemy.speech_writer.writer_text = "(sound:sans)Heh...(pc)"
await enemy.speech_writer.done
enemy.speech_bubble.visible = false
```

8) Những lưu ý khi sửa TSCN (text scene files)
- Khi chỉnh subresource script (script/source string) cẩn thận escape dấu nháy nếu sửa trực tiếp trong file .tscn.
- Thay đổi resource ids (ExtResource/ SubResource) có thể làm scene corrupt nếu không tương thích — tốt nhất sửa trong Godot editor.

9) Gợi ý code patterns để thêm animation mới cho Sans
- Thêm animation mới vào `SpriteFrames` (tên `taunt`), sau đó trong code chiêu hoặc check() gọi `head.play("taunt")`.
- Nếu muốn chi tiết, bạn có thể expose small helper function trong Enemy script, ví dụ:
```
func set_expression(name: String):
    match name:
        "smile": head.frame = 2
        "angry": head.play("angry")
        _:
            head.frame = 0
```

10) Kiểm tra & test
- Mở `res://scenes/battles/example_battles/battle_example.tscn` trong Godot
- Chạy scene (Play Scene). Quan sát `enemies/example_enemy` head/torso/legs.
- Kiểm tra Output/Debugger nếu có lỗi (missing resource, wrong field names)

11) Quick references (file paths trong repo)
- Scene: `scenes/battles/example_battles/battle_example.tscn`
- Enemy script (subresource in .tscn): GDScript_ohi4s
- Sprite images: `assets/sprites/battle/enemies/sans_undertale/`
- Writer (dialogue): `scripts/global/writer.gd`

---

Nếu bạn muốn, tôi có thể:
- Thêm đoạn code mẫu vào `scripts/battle/attacks/sans_new_attack.gd` để thay đổi face khi attack bắt đầu/kết thúc.
- Tạo một helper function trong Enemy script để set expression và gọi từ nhiều chỗ.

Bạn muốn tôi làm thêm gì?