cmake_minimum_required(VERSION 3.22)

project(KlangbildDuo VERSION 1.0.0)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# JUCE: liegt der Ordner "JUCE" neben dieser Datei, wird er benutzt,
# sonst wird JUCE automatisch von GitHub geladen.
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/JUCE/CMakeLists.txt")
    add_subdirectory(JUCE)
else()
    include(FetchContent)
    FetchContent_Declare(JUCE
        GIT_REPOSITORY https://github.com/juce-framework/JUCE.git
        GIT_TAG 8.0.4
        GIT_SHALLOW ON)
    FetchContent_MakeAvailable(JUCE)
endif()

set(KB_FORMATS VST3 Standalone)
if(APPLE)
    set(KB_FORMATS AU VST3 Standalone)
endif()

juce_add_plugin(KlangbildDuo
    COMPANY_NAME "Klangbild"
    BUNDLE_ID "de.klangbild.duo"
    PLUGIN_MANUFACTURER_CODE Klbd
    PLUGIN_CODE Kbdu
    FORMATS ${KB_FORMATS}
    PRODUCT_NAME "Klangbild Duo"
    IS_SYNTH FALSE
    NEEDS_MIDI_INPUT FALSE
    NEEDS_MIDI_OUTPUT FALSE
    IS_MIDI_EFFECT FALSE
    EDITOR_WANTS_KEYBOARD_FOCUS FALSE
    AU_MAIN_TYPE kAudioUnitType_Effect
    VST3_CATEGORIES Fx Distortion
    COPY_PLUGIN_AFTER_BUILD FALSE)

juce_add_binary_data(DuoAssets SOURCES
    Resources/Yellowtail-Regular.ttf
    Resources/Oswald-Medium.ttf
    Resources/Oswald-Regular.ttf)

# Neural Amp Modeler Core (MIT) + Eigen (MPL2) + nlohmann/json (MIT)
file(GLOB NAM_SOURCES
    "${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/NAM/*.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/NAM/*/*.cpp")
# Wichtig: NAM meldet seine Modelltypen (WaveNet, LSTM, ...) über statische Objekte an.
# Lägen diese Dateien in der statischen Shared-Code-Bibliothek von JUCE, würde der Linker
# sie als "unbenutzt" verwerfen - dann ließe sich kein Profil laden. Deshalb werden sie
# als Objektdateien direkt in jedes Plugin-Format (AU, VST3, Standalone) gelinkt.
add_library(nam_core OBJECT ${NAM_SOURCES})
set_target_properties(nam_core PROPERTIES POSITION_INDEPENDENT_CODE ON)
target_compile_options(nam_core PRIVATE -w)
target_compile_definitions(nam_core PRIVATE EIGEN_MPL2_ONLY=1)
target_include_directories(nam_core SYSTEM PRIVATE
    ThirdParty ThirdParty/NAM ThirdParty/nlohmann ThirdParty/eigen)
if(APPLE)
    set_target_properties(nam_core PROPERTIES OSX_ARCHITECTURES "${CMAKE_OSX_ARCHITECTURES}")
endif()

target_sources(KlangbildDuo PRIVATE
    Source/PluginProcessor.cpp
    Source/PluginEditor.cpp
    Source/VintageUI.cpp)

foreach(fmt ${KB_FORMATS})
    if(TARGET KlangbildDuo_${fmt})
        target_sources(KlangbildDuo_${fmt} PRIVATE $<TARGET_OBJECTS:nam_core>)
    endif()
endforeach()

target_include_directories(KlangbildDuo PRIVATE Source)
target_include_directories(KlangbildDuo SYSTEM PRIVATE
    ThirdParty
    ThirdParty/NAM
    ThirdParty/nlohmann
    ThirdParty/AudioDSPTools
    ThirdParty/eigen)

target_compile_definitions(KlangbildDuo PUBLIC
    JUCE_WEB_BROWSER=0
    JUCE_USE_CURL=0
    JUCE_VST3_CAN_REPLACE_VST2=0
    JUCE_DISPLAY_SPLASH_SCREEN=0
    EIGEN_MPL2_ONLY=1)

target_link_libraries(KlangbildDuo
    PRIVATE
        DuoAssets
        juce::juce_audio_utils
        juce::juce_dsp
    PUBLIC
        juce::juce_recommended_config_flags
        juce::juce_recommended_lto_flags
        juce::juce_recommended_warning_flags)
