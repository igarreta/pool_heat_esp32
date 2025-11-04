# Phase 7 Implementation Plan: Critical Event Notifications

**Date:** 2025-11-04
**Status:** Planning
**Goal:** Add Pushover notifications for critical system failures

---

## Overview

Implement notification system that alerts user of critical failures via Pushover, using Home Assistant as intermediary to simplify ESP32 code and improve reliability.

---

## Architecture Decision

**Selected Approach:** ESP32 → Home Assistant → Pushover

**Why not direct Pushover API from ESP32:**
- ❌ Requires HTTPS implementation on ESP32
- ❌ Requires storing Pushover API token on ESP32
- ❌ More complex code
- ❌ Depends on internet connectivity (WiFi issues block notifications)

**Benefits of HA intermediary:**
- ✅ Works over local network (WiFi issues don't block local ESP32→HA communication)
- ✅ Simpler ESP32 code (just call HA service)
- ✅ HA handles internet connectivity to Pushover independently
- ✅ Easy to change notification service later (Telegram, email, etc.)
- ✅ Already connected via HA API

---

## Requirements

### Critical Events to Notify

| Event | Current Code Location | Message | Priority |
|-------|----------------------|---------|----------|
| Sensor range violation (water) | esp32-pileta.yaml:268-276 | "⚠️ Pool: Sensor agua fuera de rango (XX°C)" | 0 |
| Sensor range violation (heater) | esp32-pileta.yaml:277-284 | "⚠️ Pool: Sensor calefactor fuera de rango (XX°C)" | 0 |
| Sensor staleness (water) | esp32-pileta.yaml:286-295 | "⚠️ Pool: Sensor agua sin actualizar (>5 min)" | 0 |
| Sensor staleness (heater) | esp32-pileta.yaml:296-304 | "⚠️ Pool: Sensor calefactor sin actualizar (>5 min)" | 0 |
| Pump desync | esp32-pileta.yaml:306-314 | "⚠️ Pool: Desincronización de bombas detectada" | 0 |
| Runtime watchdog | esp32-pileta.yaml:316-325 | "⚠️ Pool: Límite de 8 horas alcanzado" | 0 |
| Parameter validation (IMX) | esp32-pileta.yaml:146-154 | "⚠️ Pool: IMX auto-corregido (IMX ajustado a XX°C)" | 0 |
| Parameter validation (IMI) | esp32-pileta.yaml:172-177 | "⚠️ Pool: IMX auto-corregido por cambio en IMI (IMX ajustado a XX°C)" | 0 |

### Notification Constraints

- **Throttling:** Each notification type sent MAX once per day
- **Priority:** 0 (normal, not emergency)
- **Reset:** All throttle flags reset at midnight
- **No repeats:** Do not notify on same error multiple times in same day

### Events NOT to Notify

- ❌ WiFi disconnection (notifications wouldn't work anyway)
- ❌ Normal operation (heating on/off, skimmer cycles)
- ❌ Sensor readings (continuous data)
- ❌ Control logic evaluations (every 30s checks)

---

## Implementation Steps

### Step 1: Add Notification Throttle Globals

**Location:** After existing globals section (after line 261 in esp32-pileta.yaml)

**Add 8 new globals:**
```yaml
# Notification throttle flags (reset at midnight)
- id: notif_sensor_water_range_sent
  type: bool
  restore_value: no
  initial_value: 'false'

- id: notif_sensor_heater_range_sent
  type: bool
  restore_value: no
  initial_value: 'false'

- id: notif_sensor_water_stale_sent
  type: bool
  restore_value: no
  initial_value: 'false'

- id: notif_sensor_heater_stale_sent
  type: bool
  restore_value: no
  initial_value: 'false'

- id: notif_pump_desync_sent
  type: bool
  restore_value: no
  initial_value: 'false'

- id: notif_watchdog_sent
  type: bool
  restore_value: no
  initial_value: 'false'

- id: notif_param_validation_sent
  type: bool
  restore_value: no
  initial_value: 'false'

- id: notif_test_available
  type: bool
  restore_value: no
  initial_value: 'false'
```

**Purpose:** Track which notifications have been sent today to prevent repeats

---

### Step 2: Reset Notification Flags at Midnight

**Location:** Modify existing midnight reset (line 54-58)

**Before:**
```yaml
- lambda: |-
    ESP_LOGI("runtime_tracker", "Reset diario: %.2f horas registradas hoy", id(bomba_horas_hoy));
    id(bomba_horas_ayer) = id(bomba_horas_hoy);
    ESP_LOGI("runtime_tracker", "Horas ayer almacenadas: %.2f", id(bomba_horas_ayer));
    id(bomba_horas_hoy) = 0.0;
```

**After:**
```yaml
- lambda: |-
    ESP_LOGI("runtime_tracker", "Reset diario: %.2f horas registradas hoy", id(bomba_horas_hoy));
    id(bomba_horas_ayer) = id(bomba_horas_hoy);
    ESP_LOGI("runtime_tracker", "Horas ayer almacenadas: %.2f", id(bomba_horas_ayer));
    id(bomba_horas_hoy) = 0.0;

    // Reset notification throttle flags
    id(notif_sensor_water_range_sent) = false;
    id(notif_sensor_heater_range_sent) = false;
    id(notif_sensor_water_stale_sent) = false;
    id(notif_sensor_heater_stale_sent) = false;
    id(notif_pump_desync_sent) = false;
    id(notif_watchdog_sent) = false;
    id(notif_param_validation_sent) = false;
    ESP_LOGI("notifications", "Flags de notificación reseteados");
```

---

### Step 3: Add Test Button

**Location:** New `button` section (after switches section, around line 635)

**Add:**
```yaml
button:
  - platform: template
    name: "Test Pushover Notification"
    id: test_pushover_button
    icon: "mdi:bell-ring"
    on_press:
      then:
        - lambda: |-
            if (!id(notif_test_available)) {
              ESP_LOGI("notifications", "Test button pressed - sending test notification");
              id(notif_test_available) = true;
            } else {
              ESP_LOGW("notifications", "Test notification already sent today");
              return;
            }
        - homeassistant.service:
            service: notify.pushover
            data:
              title: "Pool System Test"
              message: "Test notification from ESP32-pileta. System operational."
              data:
                priority: 0
```

**Purpose:** Allow manual testing of notification path

---

### Step 4: Add Notifications to Sensor Range Violations

**Location:** Modify lines 268-284 (sensor range checks)

**Water sensor (before line 276 - turn_off action):**
```yaml
if (temp_agua < 0.0 || temp_agua > 50.0) {
  ESP_LOGE("safety", "Temperatura agua fuera de rango: %.1f°C", temp_agua);
  if (calefaccion_on) {
    ESP_LOGE("safety", "Apagando calefacción por sensor agua inválido");
    id(pileta_calefaccion_completa).turn_off();
  }

  // Send notification (once per day)
  if (!id(notif_sensor_water_range_sent)) {
    id(notif_sensor_water_range_sent) = true;
    auto call = id(homeassistant_api).get_service("notify", "pushover");
    call.set_variable("title", "Pool System Alert");
    call.set_variable("message", "⚠️ Pool: Sensor agua fuera de rango (" + String(temp_agua) + "°C)");
    call.set_variable("data", "{\"priority\": 0}");
    call.perform();
  }

  return;
}
```

**Heater sensor (before line 284 - turn_off action):**
```yaml
if (temp_calefactor < 0.0 || temp_calefactor > 80.0) {
  ESP_LOGE("safety", "Temperatura calefactor fuera de rango: %.1f°C", temp_calefactor);
  if (calefaccion_on) {
    ESP_LOGE("safety", "Apagando calefacción por sensor calefactor inválido");
    id(pileta_calefaccion_completa).turn_off();
  }

  // Send notification (once per day)
  if (!id(notif_sensor_heater_range_sent)) {
    id(notif_sensor_heater_range_sent) = true;
    auto call = id(homeassistant_api).get_service("notify", "pushover");
    call.set_variable("title", "Pool System Alert");
    call.set_variable("message", "⚠️ Pool: Sensor calefactor fuera de rango (" + String(temp_calefactor) + "°C)");
    call.set_variable("data", "{\"priority\": 0}");
    call.perform();
  }

  return;
}
```

---

### Step 5: Add Notifications to Sensor Staleness

**Location:** Modify lines 286-304 (staleness checks)

**Water sensor staleness (around line 295):**
```yaml
if (id(temp_agua_last_update) > 0 &&
    (current_time - id(temp_agua_last_update)) > 300000) {
  ESP_LOGE("safety", "Sensor agua sin actualizar por >5 min");
  if (calefaccion_on) {
    ESP_LOGE("safety", "Apagando calefacción por sensor agua obsoleto");
    id(pileta_calefaccion_completa).turn_off();
  }

  // Send notification (once per day)
  if (!id(notif_sensor_water_stale_sent)) {
    id(notif_sensor_water_stale_sent) = true;
    auto call = id(homeassistant_api).get_service("notify", "pushover");
    call.set_variable("title", "Pool System Alert");
    call.set_variable("message", "⚠️ Pool: Sensor agua sin actualizar (>5 min)");
    call.set_variable("data", "{\"priority\": 0}");
    call.perform();
  }

  return;
}
```

**Heater sensor staleness (around line 304):**
```yaml
if (id(temp_calefactor_last_update) > 0 &&
    (current_time - id(temp_calefactor_last_update)) > 300000) {
  ESP_LOGE("safety", "Sensor calefactor sin actualizar por >5 min");
  if (calefaccion_on) {
    ESP_LOGE("safety", "Apagando calefacción por sensor calefactor obsoleto");
    id(pileta_calefaccion_completa).turn_off();
  }

  // Send notification (once per day)
  if (!id(notif_sensor_heater_stale_sent)) {
    id(notif_sensor_heater_stale_sent) = true;
    auto call = id(homeassistant_api).get_service("notify", "pushover");
    call.set_variable("title", "Pool System Alert");
    call.set_variable("message", "⚠️ Pool: Sensor calefactor sin actualizar (>5 min)");
    call.set_variable("data", "{\"priority\": 0}");
    call.perform();
  }

  return;
}
```

---

### Step 6: Add Notification to Pump Desync

**Location:** Modify lines 306-314 (pump desync check)

**Before line 314 - return statement:**
```yaml
if (calefaccion_on && (!id(pileta_bomba_esp).state || !id(pileta_calefaccion_esp).state)) {
  ESP_LOGE("safety", "Desincronización detectada! Bomba=%s, Calefactor=%s",
           id(pileta_bomba_esp).state ? "ON" : "OFF",
           id(pileta_calefaccion_esp).state ? "ON" : "OFF");
  ESP_LOGE("safety", "Apagando sistema completo");
  id(pileta_calefaccion_completa).turn_off();

  // Send notification (once per day)
  if (!id(notif_pump_desync_sent)) {
    id(notif_pump_desync_sent) = true;
    auto call = id(homeassistant_api).get_service("notify", "pushover");
    call.set_variable("title", "Pool System Alert");
    call.set_variable("message", "⚠️ Pool: Desincronización de bombas detectada");
    call.set_variable("data", "{\"priority\": 0}");
    call.perform();
  }

  return;
}
```

---

### Step 7: Add Notification to Watchdog Timer

**Location:** Modify lines 316-325 (watchdog check)

**Before line 325 - return statement:**
```yaml
if (calefaccion_on && id(heating_start_time) > 0) {
  unsigned long runtime = current_time - id(heating_start_time);
  if (runtime > 28800000) {
    ESP_LOGE("safety", "Watchdog: 8 horas de operación alcanzadas");
    ESP_LOGE("safety", "Apagando por límite de tiempo independiente");
    id(pileta_calefaccion_completa).turn_off();

    // Send notification (once per day)
    if (!id(notif_watchdog_sent)) {
      id(notif_watchdog_sent) = true;
      auto call = id(homeassistant_api).get_service("notify", "pushover");
      call.set_variable("title", "Pool System Alert");
      call.set_variable("message", "⚠️ Pool: Límite de 8 horas alcanzado");
      call.set_variable("data", "{\"priority\": 0}");
      call.perform();
    }

    return;
  }
}
```

---

### Step 8: Add Notifications to Parameter Validation

**Location:** Modify lines 146-154 and 172-177 (IMX validation)

**IMX change validation (around line 154):**
```yaml
if (new_max_diff < min_allowed) {
  ESP_LOGW("safety", "IMX (%.1f) < IMI + 1 (%.1f). Forzando IMX = %.1f",
           new_max_diff, min_allowed, min_allowed);
  id(temp_max_diff) = min_allowed;

  // Send notification (once per day)
  if (!id(notif_param_validation_sent)) {
    id(notif_param_validation_sent) = true;
    auto call = id(homeassistant_api).get_service("notify", "pushover");
    call.set_variable("title", "Pool System Alert");
    call.set_variable("message", "⚠️ Pool: IMX auto-corregido (ajustado a " + String(min_allowed) + "°C)");
    call.set_variable("data", "{\"priority\": 0}");
    call.perform();
  }
} else {
  id(temp_max_diff) = new_max_diff;
}
```

**IMI change validation (around line 177):**
```yaml
if (current_max_diff < min_allowed_max) {
  ESP_LOGW("safety", "IMX (%.1f) < IMI + 1 (%.1f). Forzando IMX = %.1f",
           current_max_diff, min_allowed_max, min_allowed_max);
  id(temp_max_diff) = min_allowed_max;
  ESP_LOGI("config", "Diferencia máxima ajustada a: %.1f°C", id(temp_max_diff));

  // Send notification (once per day)
  if (!id(notif_param_validation_sent)) {
    id(notif_param_validation_sent) = true;
    auto call = id(homeassistant_api).get_service("notify", "pushover");
    call.set_variable("title", "Pool System Alert");
    call.set_variable("message", "⚠️ Pool: IMX auto-corregido por cambio en IMI (ajustado a " + String(min_allowed_max) + "°C)");
    call.set_variable("data", "{\"priority\": 0}");
    call.perform();
  }
}
```

---

## Testing Plan

### Phase 1: Test Button Validation
1. Deploy code with test button only
2. Press test button in HA UI
3. Verify Pushover notification received
4. Verify second press doesn't send (throttle works)
5. Wait for midnight, verify flag resets

### Phase 2: Critical Event Testing
1. Deploy full notification code
2. Test sensor range notification (temporarily set invalid range)
3. Test parameter validation notification (set IMX < IMI + 1)
4. Monitor logs to verify throttling works
5. Verify no duplicate notifications same day

### Phase 3: Production Monitoring
1. Deploy to production
2. Monitor logs for 48 hours
3. Verify notifications arrive when expected
4. Verify no notification spam
5. Document any issues

---

## Code Complexity Estimate

- **New globals:** 8 notification flags
- **Modified sections:** 8 (midnight reset + 7 critical error points)
- **New section:** 1 test button
- **Total additions:** ~150 lines
- **Risk level:** LOW (no changes to control logic, only adds notifications)

---

## Prerequisites

- ✅ Home Assistant has Pushover integration configured
- ✅ Service `notify.pushover` is functional
- ✅ Test with button before deploying to critical errors

---

## Rollback Plan

If notifications cause issues:
1. Comment out all `homeassistant.service` calls
2. Keep throttle globals and test button for future use
3. System continues normal operation without notifications
4. Debug notification issues separately

---

## Next Steps

1. Review this plan with user
2. Implement Step 1-3 (globals, reset, test button)
3. Deploy and test button functionality
4. Implement Steps 4-8 (critical notifications)
5. Deploy and monitor for 48 hours
6. Update PROJECT_STATUS.md when complete
