---
config:
  layout: elk
  theme: redux-dark
---
stateDiagram
  direction TB
  [*] --> READY:製造オーダー発行
  READY --> PRINTING:1個体DL開始
  PRINTING --> CUTTING
  CUTTING --> SANDING
  SANDING --> INSPECTION
  INSPECTION --> COMPLETED:合格
  INSPECTION --> SANDING:修正可能(戻り)
  INSPECTION --> DISCARD:修正困難
  COMPLETED --> SHIPPED:組立投入
  PRINTING --> DISCARD:造形失敗
  CUTTING --> DISCARD:切削失敗
  SANDING --> DISCARD:研磨失敗
  SHIPPED --> DB
  DISCARD --> DB
  INSPECTION --> CUTTING:修正可能
  READY:READY (Box確保済)
  PRINTING:PRINTING (DL実行)
  INSPECTION:INSPECTION (検査)
  COMPLETED:COMPLETED (合格)
  DISCARD:DISCARD (Box解放)
  SHIPPED:SHIPPED (Box解放)
  DB:実績DB (s4)
  note right of DISCARD 
  どの工程からでも遷移可能
      遷移時に自動でBox解放
  end note