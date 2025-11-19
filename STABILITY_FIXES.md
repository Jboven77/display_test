# Critical Stability Fixes for ESP32-S3 Waveshare Display

## Issue: Watchdog Timer Crash

**Root Cause**: The `lvgl.on_ready` block was making blocking HTTP requests that took too long, causing the ESP32 watchdog to trigger a reset.

```
E (11783) task_wdt: Task watchdog got triggered
E (11783) task_wdt:  - loopTask (CPU 0)
```

## Required Changes

### 1. Fix `on_ready` Block (CRITICAL)

**REMOVE these lines from `lvgl.on_ready`:**
```yaml
# DON'T DO THIS IN on_ready:
- script.execute: fetch_all_plug_info  # <-- CAUSES CRASH
```

The `on_ready` should ONLY:
- Show the main page
- Register activity
- Set initial clock text
- Restore persisted settings (quick operations only)

**DO NOT** make any HTTP requests in `on_ready`.

### 2. Fix Malformed YAML

Your original YAML had these syntax errors:

```yaml
# WRONG - this line is orphaned:
lv_label_set_text(id(lbl_plug8_text), "PLUG 8");

# WRONG - incomplete lambda block:
- lambda: |-
    // Restore persisted timer settings...
```

### 3. Reduce HTTP Timeout

Change:
```yaml
http_request:
  id: http_client
  timeout: 2s  # <-- TOO LONG
```

To:
```yaml
http_request:
  id: http_client
  timeout: 300ms  # Much faster, prevents blocking
```

### 4. Delay Initial Plug Info Fetch

Change the first `fetch_all_plug_info` interval from:
```yaml
- interval: 10s
  then:
    - script.execute: bl_auto_off_check
    - script.execute: fetch_all_plug_info  # <-- Runs too early
```

To:
```yaml
- interval: 10s
  then:
    - script.execute: bl_auto_off_check
    # DON'T call fetch_all_plug_info here

# Add a NEW one-time startup delay:
- interval: 30s  # Wait 30s after boot
  then:
    - if:
        condition:
          wifi.connected:
        then:
          - script.execute: fetch_all_plug_info
          - delay: 1s
```

## UI Changes Requested

### 1. Change Timer Button Labels

**Main Page Timer Buttons** - Change from "Timer OFF" to just "Timer":

```yaml
# OLD:
widgets:
  - label:
      id: lbl1_timer_main
      text: "Timer OFF"

# NEW:
widgets:
  - label:
      text: "Timer"  # Simple, static text
```

### 2. Change Schedule Toggle Labels

**Timer Pages** - Change "Schedule: ON/OFF" to "Timer: ON/OFF":

```yaml
# In switch definitions, change:
lv_label_set_text(lbl, next ? "Schedule: ON" : "Schedule: OFF");

# To:
lv_label_set_text(lbl, next ? "Timer: ON" : "Timer: OFF");
```

### 3. Make Timer Buttons Update Colors

Add `on_turn_on` and `on_turn_off` handlers to each schedule switch:

```yaml
switch:
  - platform: template
    id: plug1_schedule_enabled
    name: "Plug 1 Schedule Enabled"
    optimistic: true
    entity_category: "config"
    on_turn_on:
      - lambda: |-
          id(plug1_schedule_persist) = true;
          lv_label_set_text(id(lbl_schedule1_state), "Timer: ON");
          lv_obj_set_style_bg_color(id(btn_plug1_timer), lv_color_hex(0x228B22), LV_PART_MAIN);
    on_turn_off:
      - lambda: |-
          id(plug1_schedule_persist) = false;
          lv_label_set_text(id(lbl_schedule1_state), "Timer: OFF");
          lv_obj_set_style_bg_color(id(btn_plug1_timer), lv_color_hex(0x444444), LV_PART_MAIN);
```

### 4. Persist Timer Values on Change

Add `on_value` handlers to number and select components:

```yaml
number:
  - platform: template
    id: plug1_on_hour
    # ... other config ...
    on_value:
      - lambda: |-
          id(plug1_on_hour_persist) = (int)x;

select:
  - platform: template
    id: plug1_on_ampm
    # ... other config ...
    on_value:
      - lambda: |-
          id(plug1_on_ampm_persist) = x;
```

## Complete Fixed on_ready Block

```yaml
lvgl:
  on_ready:
    then:
      - lvgl.page.show:
          id: main_page
      - script.execute: register_activity
      - lambda: |-
          // Set initial clock display ONLY
          auto now = id(sntp_time).now();
          if (!now.is_valid()) return;
          char buf[16];
          int h12 = now.hour % 12;
          if (h12 == 0) h12 = 12;
          bool pm = now.hour >= 12;
          sprintf(buf, "%02d:%02d %s", h12, now.minute, pm ? "PM" : "AM");
          lv_label_set_text(id(lbl_main_time), buf);
          lv_label_set_text(id(lbl_timer1_clock), buf);
          lv_label_set_text(id(lbl_timer2_clock), buf);
          lv_label_set_text(id(lbl_timer3_clock), buf);
          lv_label_set_text(id(lbl_timer4_clock), buf);
          lv_label_set_text(id(lbl_timer5_clock), buf);
          lv_label_set_text(id(lbl_timer6_clock), buf);
          lv_label_set_text(id(lbl_timer7_clock), buf);
          lv_label_set_text(id(lbl_timer8_clock), buf);

          // Restore persisted settings (FAST operations only)
          bool e1 = id(plug1_schedule_persist);
          id(plug1_schedule_enabled).publish_state(e1);
          id(plug1_on_hour).publish_state((int)id(plug1_on_hour_persist));
          id(plug1_on_minute).publish_state((int)id(plug1_on_minute_persist));
          id(plug1_off_hour).publish_state((int)id(plug1_off_hour_persist));
          id(plug1_off_minute).publish_state((int)id(plug1_off_minute_persist));
          id(plug1_on_ampm).publish_state(id(plug1_on_ampm_persist));
          id(plug1_off_ampm).publish_state(id(plug1_off_ampm_persist));
          lv_label_set_text(id(lbl_schedule1_state), e1 ? "Timer: ON" : "Timer: OFF");
          lv_obj_set_style_bg_color(id(btn_plug1_timer), e1 ? lv_color_hex(0x228B22) : lv_color_hex(0x444444), LV_PART_MAIN);

          // Repeat for plugs 2-8...
          bool e2 = id(plug2_schedule_persist);
          id(plug2_schedule_enabled).publish_state(e2);
          id(plug2_on_hour).publish_state((int)id(plug2_on_hour_persist));
          id(plug2_on_minute).publish_state((int)id(plug2_on_minute_persist));
          id(plug2_off_hour).publish_state((int)id(plug2_off_hour_persist));
          id(plug2_off_minute).publish_state((int)id(plug2_off_minute_persist));
          id(plug2_on_ampm).publish_state(id(plug2_on_ampm_persist));
          id(plug2_off_ampm).publish_state(id(plug2_off_ampm_persist));
          lv_label_set_text(id(lbl_schedule2_state), e2 ? "Timer: ON" : "Timer: OFF");
          lv_obj_set_style_bg_color(id(btn_plug2_timer), e2 ? lv_color_hex(0x228B22) : lv_color_hex(0x444444), LV_PART_MAIN);

          // ... Continue for plugs 3-8
```

## Testing Steps

After applying fixes:

1. Flash the device
2. Monitor logs - should NOT see watchdog errors
3. Device should boot successfully and show main page
4. Timer buttons should show correct colors (green=active, grey=inactive)
5. Toggle a timer - button color should change immediately
6. Timer pages should show "Timer: ON" or "Timer: OFF"

## Summary

The critical fix is: **DO NOT make HTTP requests in `on_ready`**. Move all network operations to intervals that run AFTER the device has fully booted (30+ seconds).
