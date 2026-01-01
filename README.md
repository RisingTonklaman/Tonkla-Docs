# TonklaEngine Rules (trans_rack_moves)

## แนวคิดหลัก

trans_rack_moves คือแหล่งข้อมูลจริงของการเคลื่อนที่ rack และกฎจะตัดสินว่า
ตำแหน่งนั้นถูกครอบครองหรือถูก lock จากค่า status_process

## ความหมายของสถานะ

- status_process = 4, 48
  - rack อยู่นิ่งที่ current_position (ถึงจุดหมายแล้ว/พร้อมทำงานต่อ)
  - กฎ: to_position_id ต้องเท่ากับ current_position
- status_process = 1, 2
  - rack กำลังย้ายจาก from_position_id -> to_position_id
  - กฎ: current_position ต้องเท่ากับ from_position_id
- status_process = 49
  - rack ไม่อยู่ในระบบ; ไม่ถือว่าครอบครองตำแหน่งใด

หมายเหตุ: กฎทั้งหมดใน TonklaEngine ใช้เฉพาะ status_process เท่านั้น (ไม่ใช้ status_task)

## กฎตำแหน่ง

Occupied (idle):

- current_position จะถูกครอบครองเมื่อ status_process เป็น 4 หรือ 48

Locked (moving):

- from_position_id และ to_position_id จะถูก lock เมื่อ status_process เป็น 1 หรือ 2

## การทำงาน

bindPodAndBerth

- bind = 1: ห้ามทำบนตำแหน่งที่ถูกครอบครองหรือถูก lock
- bind = 0: ทำได้บนตำแหน่งที่ถูกครอบครอง แต่ห้ามบนตำแหน่งที่ถูก lock

genAgvSchedulingTask

- From (start):
  - ห้ามถ้า "locked" โดย rack ที่กำลังเคลื่อนที่
  - ทำได้แม้ "occupied" (idle)
  - ถ้ามี podCode และ status_process ของ rack นั้นเป็น 4/48,
    From ต้องเท่ากับ current_position ของ rack นั้น
- To (destination):
  - ห้ามถ้า "occupied" หรือ "locked"

## TonklaEngine2 (กฎ type)

- fromType/toType ต้องเป็น: 00, 03, 04
- toType ห้ามเป็น 03
- order_by ต้องเป็น: ASC หรือ DESC (ถ้าไม่ส่งมา default = ASC)
- fromType = 00:
  - ถ้าไม่มี podCode ให้หา podCode จาก From ผ่าน queryPodBerthAndMat
- fromType = 03:
  - From คือ podCode
  - ถ้ามี podCode ใน request ต้องตรงกับ From
  - หา From ที่แท้จริงจาก queryPodBerthAndMat (podCode -> positionCode)
  - ตอนยิง RCS ให้ใช้ type 00 สำหรับ From
- fromType = 04:
  - From คือ area_name (เช่น Cr_1)
  - หา master_positions ด้วย area_name แล้วเรียงตาม runno ด้วย order_by
  - ใช้ master_positions.id เทียบกับ queryPodBerthAndMat.positionCode
  - ต้องมีอย่างน้อย 1 podCode ใน area นี้
  - ถ้ามีหลายตัว ให้เลือกตัวแรกตามลำดับ runno (ASC/DESC)
  - ถ้ามี podCode ใน request ต้องตรงกับตัวที่เลือก
  - ตอนยิง RCS ให้ใช้ type 00 และส่ง podCode ที่เลือก
- toType = 04:
  - To คือ area_name (เช่น Cr_1)
  - หา master_positions ด้วย area_name แล้วเรียงตาม runno ด้วย order_by
  - ใช้ master_positions.id เทียบกับ queryPodBerthAndMat.positionCode
  - ต้องมีอย่างน้อย 1 ตำแหน่งที่ว่าง (ไม่มี podCode)
  - ถ้ามีหลายตำแหน่งว่าง ให้เลือกตัวแรกตามลำดับ runno (ASC/DESC)
  - ตอนยิง RCS ให้ใช้ type 00 และส่ง destination เป็นตำแหน่งที่เลือก

## TonklaEngine1 (กฎระดับ rack)

- ถ้า rack_id มี status_process = 1 หรือ 2:
  - ห้ามใช้ genAgvSchedulingTask ที่เกี่ยวกับ rack_id นั้น
  - ห้ามใช้ bindPodAndBerth ที่เกี่ยวกับ rack_id นั้น
- ถ้า status_process = 4 หรือ 48:
  - ใช้ robot-worker ได้ตามกฎเดิม
- ถ้า status_process = 49:
  - ใช้ robot-worker ได้ทุกกฎ

## TonklaEngine3 (กฎการ cancel)

- cancel ทำด้วย rack_id จาก trans_rack_moves
- status_process = 4, 48, 49:
  - ห้าม cancel
- status_process = 1:
  - cancel แล้ว status_process = 48
  - from_position_id, to_position_id, current_position จะเป็นค่าเดิมของ from_position_id
  - trans_rack_material.material_types_id จะเป็น unknow_mat
- status_process = 2:
  - cancel แล้ว status_process = 49
  - from_position_id, to_position_id, current_position จะเป็นค่าเดิมของ from_position_id
  - trans_rack_material.material_types_id จะเป็น unknow_mat

## TonklaEngine4 (กฎการเปลี่ยน rack/material)

- ทุกครั้งที่ bindPodAndBerth สำเร็จ (indBind = 1 หรือ 0)
  - trans_rack_material.material_types_id ต้องเป็น unknow_mat
- ทุกครั้งที่มีการเปลี่ยน material_types_id ใน trans_rack_material:
  - ห้ามเปลี่ยนถ้า trans_rack_moves.status_process = 1 หรือ 2
  - ยกเว้นกรณีที่เปลี่ยนจากกฎ TonklaEngine3 (cancel)

## การตรวจความสอดคล้องของข้อมูล

เมื่ออ่าน trans_rack_moves เพื่อใช้ในการ lock ระบบจะตรวจ:

- idle: to_position_id == current_position
- moving: current_position == from_position_id

ถ้ากฎผิด ระบบจะ throw error (BadRequest/Conflict)
