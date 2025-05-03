README.md

````markdown
# C++ Game Boy Emulator (Gambatte + SDL2)

This repo implements a fully working Game Boy emulator in C++ using the Gambatte core and SDL2 for graphics, audio, and input.

## Requirements
- C++17 compiler (GCC/Clang/MSVC)
- CMake 3.15+
- SDL2 development libraries
- Git

## Setup
```bash
# Clone repository and submodule
git clone https://github.com/your_username/gb_emulator_cpp.git
cd gb_emulator_cpp
git submodule update --init --recursive
````

## Build

```bash
mkdir build && cd build
cmake ..
cmake --build .
```

## Run

```bash
# From build directory
./gb_emulator_cpp /path/to/rom.gb
```

## Controls

* Arrow Keys: D-pad
* Z: Button A
* X: Button B
* Enter: Start
* Right Shift: Select
* Esc: Quit

````

---

CMakeLists.txt
```cmake
cmake_minimum_required(VERSION 3.15)
project(gb_emulator_cpp)
set(CMAKE_CXX_STANDARD 17)

# Gambatte core (submodule)
add_subdirectory(externals/gambatte)

# SDL2
find_package(SDL2 REQUIRED)
include_directories(${SDL2_INCLUDE_DIRS})

# Source files
data_directory(
    TARGET gb_emulator_cpp
    BASE_DIR ${CMAKE_SOURCE_DIR}/data
)
add_executable(gb_emulator_cpp src/main.cpp)

target_link_libraries(gb_emulator_cpp PRIVATE gambatte ${SDL2_LIBRARIES})

# Install binary to dist/
install(TARGETS gb_emulator_cpp DESTINATION dist)
```}

---

src/main.cpp
```cpp
#include <iostream>
#include <vector>
#include <SDL.h>
#include "gambatte.h"

const int SCREEN_WIDTH = 160 * 3;
const int SCREEN_HEIGHT = 144 * 3;

// Audio callback buffer
audio_callback_t audioCallback;

void audio_callback(void* userdata, Uint8* stream, int len) {
    // Pull samples from the emulator's audio buffer
    audioCallback(stream, len);
}

int main(int argc, char* argv[]) {
    if (argc < 2) {
        std::cerr << "Usage: " << argv[0] << " <path/to/rom.gb>" << std::endl;
        return 1;
    }
    const char* romPath = argv[1];

    // Initialize SDL
    if (SDL_Init(SDL_INIT_VIDEO | SDL_INIT_AUDIO | SDL_INIT_EVENTS) < 0) {
        std::cerr << "SDL Init Error: " << SDL_GetError() << std::endl;
        return 1;
    }

    SDL_Window* window = SDL_CreateWindow(
        "C++ Game Boy Emulator",
        SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED,
        SCREEN_WIDTH, SCREEN_HEIGHT,
        SDL_WINDOW_SHOWN
    );
    SDL_Renderer* renderer = SDL_CreateRenderer(window, -1, SDL_RENDERER_ACCELERATED);
    SDL_Texture* texture = SDL_CreateTexture(
        renderer,
        SDL_PIXELFORMAT_RGB24,
        SDL_TEXTUREACCESS_STREAMING,
        160, 144
    );

    // Initialize Gambatte
    gambatte::GbInitials init;
    gambatte::Gambatte *emu = gambatte::createGBMachine(init);
    emu->loadROM(romPath);

    // Setup audio spec
    SDL_AudioSpec want{}, have{};
    want.freq = 44100;
    want.format = AUDIO_S16SYS;
    want.channels = 2;
    want.samples = 1024;
    want.callback = audio_callback;
    want.userdata = emu;

    audioCallback = [&](void* stream, int len) {
        emu->runTill(emu->cpuTime() + (len/4));
        // Gambatte writes audio to its internal buffer
        int16_t* buf = reinterpret_cast<int16_t*>(stream);
        int samples = len / sizeof(int16_t);
        emu->copyAudioSamples(buf, samples);
    };

    if (SDL_OpenAudio(&want, &have) < 0) {
        std::cerr << "SDL Audio Error: " << SDL_GetError() << std::endl;
    }
    SDL_PauseAudio(0);

    // Main loop
    bool running = true;
    SDL_Event e;
    std::vector<uint32_t> pixels(160 * 144);

    while (running) {
        while (SDL_PollEvent(&e)) {
            if (e.type == SDL_QUIT) running = false;
            if (e.type == SDL_KEYDOWN || e.type == SDL_KEYUP) {
                bool down = (e.type == SDL_KEYDOWN);
                switch (e.key.keysym.sym) {
                    case SDLK_z:    emu->setJoyp(0x01, down); break; // A
                    case SDLK_x:    emu->setJoyp(0x02, down); break; // B
                    case SDLK_RETURN: emu->setJoyp(0x08, down); break; // Start
                    case SDLK_RSHIFT: emu->setJoyp(0x04, down); break; // Select
                    case SDLK_UP:   emu->setJoyp(0x10, down); break;
                    case SDLK_DOWN: emu->setJoyp(0x20, down); break;
                    case SDLK_LEFT: emu->setJoyp(0x40, down); break;
                    case SDLK_RIGHT: emu->setJoyp(0x80, down); break;
                    case SDLK_ESCAPE: running = false; break;
                }
            }
        }

        // Run one frame and get pixels
        emu->stepForFrame();
        const uint_least32_t* frame = emu->frameBuffer();
        for (int i = 0; i < 160 * 144; ++i) pixels[i] = frame[i] | 0xFF000000;

        // Update texture & render
        SDL_UpdateTexture(texture, nullptr, pixels.data(), 160 * sizeof(uint32_t));
        SDL_RenderClear(renderer);
        SDL_RenderCopy(renderer, texture, nullptr, nullptr);
        SDL_RenderPresent(renderer);

        SDL_Delay(16);
    }

    // Cleanup
    SDL_CloseAudio();
    emu->saveStateToFile("save.sav");
    delete emu;
    SDL_DestroyTexture(texture);
    SDL_DestroyRenderer(renderer);
    SDL_DestroyWindow(window);
    SDL_Quit();

    return 0;
}
````
