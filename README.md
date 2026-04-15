# Cảm ứng Unity → ImGui trên Android (Realme)

## 1. Đoạn mã chính 

```cpp
#pragma once

#include "quoctoandevlor/IL2CppSDKGenerator/Vector2.h"
#include "ImGui/Call_ImGui.h"
#include "quoctoandevlor/Call_Me.h"
#include <mutex>
#include <cstdio>
#include <cstring>
#include <inttypes.h>

uintptr_t GetBaseTouch(const char *name) {
    uintptr_t base = 0;
    char line[512];
    FILE *f = fopen("/proc/self/maps", "r");
    if (!f) return 0;
    while (fgets(line, sizeof(line), f)) {
        uintptr_t tmpBase;
        char tmpName[256] = {0};
        if (sscanf(line, "%" PRIxPTR "-%*" PRIxPTR " %*s %*s %*s %*s %255s", &tmpBase, tmpName) >= 1) {
            if (strstr(tmpName, name)) {
                base = tmpBase;
                break;
            }
        }
    }
    fclose(f);
    return base;
}

namespace UnityEngine {
namespace Input {

enum class TouchPhase { Began, Moved, Stationary, Ended, Canceled };
enum class TouchType { Direct, Indirect, Stylus };

struct Touch {
    int m_FingerId{};
    Vector2 m_Position;
    Vector2 m_RawPosition;
    Vector2 m_PositionDelta;
    float m_TimeDelta{};
    int m_TapCount{};
    TouchPhase m_Phase;
    TouchType m_Type;
    float m_Pressure{};
    float m_maximumPossiblePressure{};
    float m_Radius{};
    float m_RadiusVariance{};
    float m_AltitudeAngle{};
    float m_AzimuthAngle{};
};

static bool is_done = false;
static std::mutex setup_mutex;

Touch (*UnityEngine_Input_GetTouch)(int index) = nullptr;
int (*UnityEngine_Input_get_touchCount)() = nullptr;

bool Done() {
    std::lock_guard<std::mutex> lock(setup_mutex);
    if (!is_done) {
        uintptr_t base = GetBaseTouch("libil2cpp.so");
        if (base == 0) return false;

        // TODO: thay RVA đúng với game của bạn
        uintptr_t rva_get_touch   = 0x9547C18;
        uintptr_t rva_touch_count = 0x9548168;

        void* getTouch = (void*)(base + rva_get_touch);
        void* getCount = (void*)(base + rva_touch_count);

        if (getTouch) UnityEngine_Input_GetTouch = (Touch (*)(int))getTouch;
        if (getCount) UnityEngine_Input_get_touchCount = (int (*)())getCount;
        is_done = true;
    }
    return (UnityEngine_Input_GetTouch && UnityEngine_Input_get_touchCount);
}

void UpdateTouchInput() {
    if (!UnityEngine_Input_GetTouch || !UnityEngine_Input_get_touchCount) {
        if (!Done()) return;
    }

    if (ImGui::GetCurrentContext() == nullptr) return;
    ImGuiIO& io = ImGui::GetIO();
    if (io.DisplaySize.x <= 0 || io.DisplaySize.y <= 0) return;

    int touchCount = UnityEngine_Input_get_touchCount();
    if (touchCount < 0 || touchCount > 10) touchCount = 0;

    for (int idx = 0; idx < touchCount && idx < 3; ++idx) {
        Touch touch = UnityEngine_Input_GetTouch(idx);
        float x = touch.m_Position.x;
        float y = round(io.DisplaySize.y) - touch.m_Position.y;

        switch (touch.m_Phase) {
            case TouchPhase::Began:
                io.MousePos = ImVec2(x, y);
                io.MouseDown[idx] = true;
                break;
            case TouchPhase::Moved:
            case TouchPhase::Stationary:
                io.MousePos = ImVec2(x, y);
                break;
            case TouchPhase::Ended:
            case TouchPhase::Canceled:
                io.MouseDown[idx] = false;
                break;
            default: break;
        }
    }
    for (int idx = touchCount; idx < 3; ++idx) io.MouseDown[idx] = false;
}

} // namespace Input
} // namespace UnityEngine

Gọi UnityEngine::Input::UpdateTouchInput() mỗi frame (sau khi ImGui khởi tạo).
Gọi UnityEngine::Input::Done(); tại thread main.
