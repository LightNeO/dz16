# DZ-16

## 1. Checklist

### 1.1 Version 1.3.0

| Check | PRD | Results | Evidence |
|---|---|---|---|
| Lamp on/off | FR-L1 | PASS | `logs/01_Lamp_on_off.log` |
| Named colors | FR-L2 | PASS | `logs/02_named_colors.log` |
| Custom RGB | FR-L3 | PASS | `logs/03_custom_rgb.log` |
| Named colors(real color matches) | FR-L4 | FAIL (BUG-01) | `images/BUG-01_1.jpg`, `images/BUG-01_2.jpg`, `images/BUG-01_3.jpg`, `images/BUG-01_4.jpg` |
| Brightness | FR-L5 | FAIL (BUG-02) | `logs/05_brightness(BUG-02).log` |
| Modes | FR-L6 | PASS | `logs/06_modes.log` |
| Timer | FR-L7 | PASS | `logs/07_lamp_timer.log` |
| Lamp JSON status | FR-L8 | PASS | `logs/08_lamp_status.log` |
| Scenes | FR-L10 | PASS: не реалізовано до 1.5.0 | `logs/10_scenes.log` |
| Boot/version/partition | OTA-1 | PASS | `logs/11_initial_boot_1.3.0.log` |
| `ota check` | OTA-2 | PASS | `logs/12_ota_check.log` |
| OTA 1.3.0 → 1.4.0 | OTA-3, OTA-5, OTA-9 | PASS | `logs/13_ota_update.log` |
| `ota install` | OTA-4 | PASS | `logs/14_ota_install.log` |
| Wifi lost during downloading | OTA-6 | FAIL (BUG-03) | `logs/15_wifi_lost(BUG-03).log` |
| No Wifi | OTA-7 | PASS | `logs/16_no_wifi.log` |
| State persistence | FR-L9, OTA-8 | FAIL (BUG-04) | `logs/17_persistence.log` |
| Rollback | OTA-10 | PASS | `logs/18_rollback.log` |


### 1.2 Version 1.4.0

Check | PRD | Results | Evidence |

### 4.3 Версія 1.5.0

Check | PRD | Results | Evidence |

## 5. OTA-8 — data save

| OTA-перехід | Wi-Fi до/після | Lamp state до/після | Scenes | Результат | Доказ |
|---|---|---|---|---|---|
| 1.3.0 → 1.4.0 | Не виконано | Не виконано | N/A | Не виконано | — |
| 1.4.0 → 1.5.0 | Не виконано | Не виконано | Перевірити на 1.5.0 | Не виконано | — |
| Forced downgrade | Не виконано | Не виконано | Перевірити за версією | Не виконано | — |

## 6. Обрив та реконект під час OTA

| Сценарій | Очікуваний результат за PRD OTA-6/OTA-7 | Фактичний результат | Статус | Доказ |
|---|---|---|---|---|
| Обрив під час 1.3.0 → 1.4.0 | Контрольована помилка, активна прошивка bootable | Не виконано | Не виконано | — |
| Повтор після відновлення Wi-Fi | OTA завершується, self-check passed | Не виконано | Не виконано | — |
| Обрив під час 1.4.0 → 1.5.0 | Контрольована помилка, активна прошивка bootable | Не виконано | Не виконано | — |
| Повтор після відновлення Wi-Fi | OTA завершується, self-check passed | Не виконано | Не виконано | — |

## 7. Bugs

### BUG-001 — Some colors names of the `lamp color <name>` command are not matched to the real device led

### BUG-002 — The `color brightness <value>` command values are not limited to the 0–100 range

### BUG-003 — The install/update process stacks after reconnecting the wifi during downloading process

### BUG-004 Wifi credentians are not saved after updating to firmware 1.5.0


Шаблон

#### BUG-001 — [Короткий заголовок]

- **Версія:**
- **Відтворюється у версіях:**
- **Regression:** так / ні
- **Severity:**
- **Priority:**
- **Відтворюваність:** 0/3
- **Передумови:**
- **Кроки:**
- **Очікуваний результат / PRD:**
- **Фактичний результат:**
- **Доказ:**

## 8. Bug matrix

| Баг | 1.3.0 | 1.4.0 | 1.5.0 | Регресія |
|---|---|---|---|---|
| BUG-001 | — | — | — | — |

## 9. Висновок

Фінальний висновок щодо переведення 1.5.0 з beta у stable буде сформульовано після завершення:

- повного lamp checklist на 1.5.0;
- OTA-8 persistence;
- interrupted OTA та retry;
- downgrade protection;
- forced downgrade;
- `ota broken` rollback;
- повторного запуску всіх обов'язкових сценаріїв.

Поточний висновок: **рішення ще не прийнято, тестування триває**.
