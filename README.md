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
- fromType = 00:
  - ถ้าไม่มี podCode ให้หา podCode จาก From ผ่าน queryPodBerthAndMat
- fromType = 03:
  - From คือ podCode
  - หา From ที่แท้จริงจาก queryPodBerthAndMat (podCode -> positionCode)
  - ตอนยิง RCS ให้ใช้ type 00 สำหรับ From
- fromType/toType = 04:
  - reserved สำหรับ area mapping (findPointFromArea), ตอนนี้ยังไม่รองรับ

## TonklaEngine1 (กฎระดับ rack)

- ถ้า rack_id มี status_process = 1 หรือ 2:
  - ห้ามใช้ genAgvSchedulingTask ที่เกี่ยวกับ rack_id นั้น
  - ห้ามใช้ bindPodAndBerth ที่เกี่ยวกับ rack_id นั้น
- ถ้า status_process = 4 หรือ 48:
  - ใช้ robot-worker ได้ตามกฎเดิม
- ถ้า status_process = 49:
  - ใช้ robot-worker ได้ทุกกฎ

## การตรวจความสอดคล้องของข้อมูล

เมื่ออ่าน trans_rack_moves เพื่อใช้ในการ lock ระบบจะตรวจ:

- idle: to_position_id == current_position
- moving: current_position == from_position_id

ถ้ากฎผิด ระบบจะ throw error (BadRequest/Conflict)
