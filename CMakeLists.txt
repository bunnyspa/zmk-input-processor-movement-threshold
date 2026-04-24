if(CONFIG_ZMK_INPUT_PROCESSOR_THRESHOLD)
  target_sources(app PRIVATE src/threshold.c)
  target_include_directories(app PRIVATE include)
endif()
