# Quy trình tạo cây từ DCC vào UE: Nanite Assemblies, Dynamic Wind

## 0. Trước khi bắt đầu…
Hướng dẫn này dành cho những người có kiến thức cao hơn chút so với người mới bắt đầu, tập trung hoàn toàn vào workflow Nanite Assemblies, Dynamic Wind — không liên quan đến việc tạo mô hình hay lưới (mesh).

## 0.5 Phần có giải thích, [QUÁ TRÌNH LÀM VÀ HỌC](https://excalidraw.com/#json=QlTYgLSA3g45wRe6X-kKc,4u87kJbNKu6bzxpqxv2rsQ) _**/còn đang cập nhật phần 1./**_
## Còn đây là phần thẳng như ruột ngựa, step by step:
## 2.0 HOUDINI 

### 2.1 [VIDEO A NÀY](https://www.youtube.com/watch?v=b_jhGZ2jto8&feature=youtu.be) giải quyết vấn đề Nanite Assembly từ Houdini -> UE
---
### 2.2 Toàn bộ node ở obj/
node nào màu xám thì kệ nó đi ha
<img width="1378" height="1134" alt="image" src="https://github.com/user-attachments/assets/db9aa1a2-d3a3-4003-b30a-09d2ce620b5e" />

### 2.3 Trong vidieo không có show 3 node cuối cho phần SK_eletal_M_esh Trunk
<img width="951" height="546" alt="image" src="https://github.com/user-attachments/assets/a6d5778a-21eb-4233-9c74-1587bb0351f4" />
<img width="962" height="669" alt="image" src="https://github.com/user-attachments/assets/5d1dded5-9d1a-44cb-bd94-01a139e322ee" />
<img width="947" height="670" alt="image" src="https://github.com/user-attachments/assets/8a79cc4a-649d-4177-aad6-22eeca723ded" />

### 2.4 Và vấn đề Orient trong Solaris
allow edit node để vào sâu hơn
<img width="1917" height="1033" alt="image" src="https://github.com/user-attachments/assets/65da2258-1a4d-482d-aeb7-738f96f882eb" />
tìm tới node **[copytopoints3]** để lấy điểm đặt lá
<img width="1480" height="1037" alt="image" src="https://github.com/user-attachments/assets/7f96ff1a-f1ed-47f6-b751-dcb016a0149d" />
<img width="1384" height="818" alt="image" src="https://github.com/user-attachments/assets/f17f8d58-b271-413c-81eb-4ccc70002be1" />
<img width="1295" height="786" alt="image" src="https://github.com/user-attachments/assets/9ae8db1c-e66a-4c24-9d3e-06d2f0e15498" />
<img width="1847" height="1047" alt="image" src="https://github.com/user-attachments/assets/83bddfbc-4754-45a3-995a-ec7dcc48d3df" />

### 2.5 Xuất JSON để áp dụng wind dynamic trong UE
<img width="1339" height="923" alt="image" src="https://github.com/user-attachments/assets/b770ffeb-11e6-4d19-8f7d-b413ac1bc175" />

<details>
	<summary>Code Python attribute2JSON</summary>
	
```import hou
import json
import os

# ----------------------------
# Config: đường dẫn file JSON xuất ra
# ----------------------------
output_path = r"C:/temp/houdini_wind.json"

# ----------------------------
# Lấy geometry của node hiện tại
# ----------------------------
node = hou.pwd()
geo = node.geometry()

# ----------------------------
# Thu thập dữ liệu từ point attributes
# ----------------------------
objects_data = []

if "boneCapture" not in [attr.name() for attr in geo.pointAttribs()]:
    raise ValueError("Attribute 'boneCapture' không tồn tại trên geo!")
if "branch_level" not in [attr.name() for attr in geo.pointAttribs()]:
    raise ValueError("Attribute 'branch_level' không tồn tại trên geo!")

for pt in geo.points():
    bone_capture = pt.attribValue("boneCapture")
    branch_level = pt.attribValue("branch_level")

    # Nếu boneCapture là tuple/list, lấy phần tử >=0 đầu tiên
    if isinstance(bone_capture, (tuple, list)):
        first_valid = next((int(x) for x in bone_capture if x >= 0), None)
        if first_valid is None:
            continue
        bone_number = first_valid
    else:
        bone_number = int(bone_capture)

    joint_name = f"joint{bone_number}"

    objects_data.append({
        "JointName": joint_name,
        "SimulationGroupIndex": branch_level
    })

# ----------------------------
# Loại bỏ duplicate joint
# ----------------------------
seen = set()
unique_objects = []
for obj in objects_data:
    key = (obj["JointName"], obj["SimulationGroupIndex"])
    if key not in seen:
        seen.add(key)
        unique_objects.append(obj)

# ----------------------------
# Tạo JSON theo format Unreal
# ----------------------------
joints = [{"JointName": "Root", "SimulationGroupIndex": 0}]

for obj in unique_objects:
    joints.append({
        "JointName": obj["JointName"],
        "SimulationGroupIndex": obj["SimulationGroupIndex"]
    })

# Tạo simulation groups dựa trên max branch_level
max_level = max(obj['SimulationGroupIndex'] for obj in unique_objects)
simulation_groups = []

for i in range(1, max_level + 1):
    if i == 1:
        simulation_groups.append({"bUseDualInfluence": False, "Influence": 1.0, "bIsTrunkGroup": True})
    else:
        min_influence = 0.2 + (i - 2) * 0.2
        max_influence = min(1.0, min_influence + 0.4)
        shift_top = max(0.0, 0.3 - (i - 2) * 0.1)
        simulation_groups.append({
            "bUseDualInfluence": True,
            "MinInfluence": round(min_influence, 2),
            "MaxInfluence": round(max_influence, 2),
            "ShiftTop": round(shift_top, 2),
            "bIsTrunkGroup": False
        })

wind_hierarchy = {
    "Joints": joints,
    "SimulationGroups": simulation_groups,
    "bIsGroundCover": False,
    "GustAttenuation": 0.25
}

# ----------------------------
# Ghi file JSON
# ----------------------------
os.makedirs(os.path.dirname(output_path), exist_ok=True)

with open(output_path, 'w', encoding='utf-8') as f:
    json.dump(wind_hierarchy, f, indent=2)

print(f"✅ JSON saved to: {output_path}")
print(f"Total joints: {len(joints)}")
print(f"Total simulation groups: {len(simulation_groups)}")
```
</details>

## 3.0 UE
## Xem tiếp [VIDEO B NÀY](https://youtu.be/3f7miRB9_Eo?si=EP3ZBYV9OJGNTFqM&t=541)
### Import JSON vào SKM 
_Config: đường dẫn file JSON xuất ra output_path = r"C:/temp/houdini_wind.json"_
<img width="1919" height="1199" alt="image" src="https://github.com/user-attachments/assets/7abe49a6-9352-4915-bb30-64f71cf31225" />
