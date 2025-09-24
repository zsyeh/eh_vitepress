# 先飞z1mini云台

**cuav的线序貌似是反的**

1.15.x按照手册配置好之后无反馈消息



后面使用6c mini尝试



| 参数组  | 参数名          | 建议设置值                   |
| ------- | --------------- | ---------------------------- |
| MAVLink | `MAV_1_CONFIG`  | `TELEM2`                     |
|         | `MAV_1_MODE`    | `Custom` 或 `Gimbal`         |
| Serial  | `SER_TEL2_BAUD` | `115200 8N1`                 |
| Mount   | `MNT_MODE_OUT`  | `MAVLink gimbal protocol v2` |

​	