idf_build_get_property(idf_target IDF_TARGET)

if(CONFIG_ESP32_NO_BLOBS OR CONFIG_ESP32S2_NO_BLOBS)
    set(link_binary_libs 0)
    set(ldfragments)
else()
    set(link_binary_libs 1)
    set(ldfragments "linker.lf")
endif()

idf_component_register(SRCS "src/phy_init.c"
                    INCLUDE_DIRS "include" "${idf_target}/include"
                    PRIV_REQUIRES nvs_flash
                    LDFRAGMENTS "${ldfragments}")

idf_build_get_property(build_dir BUILD_DIR)

set(target_name "${idf_target}")
target_link_libraries(${COMPONENT_LIB} PUBLIC "-L ${CMAKE_CURRENT_SOURCE_DIR}/lib/${target_name}")

if(link_binary_libs)
    target_link_libraries(${COMPONENT_LIB} PUBLIC phy)

    idf_component_get_property(esp_phy_lib esp_phy COMPONENT_LIB)
    target_link_libraries(${COMPONENT_LIB} INTERFACE $<TARGET_FILE:${esp_phy_lib}> libphy.a
                          $<TARGET_FILE:${esp_phy_lib}>)

    if(CONFIG_IDF_TARGET_ESP32)
        target_link_libraries(${COMPONENT_LIB} PUBLIC rtc)
    endif()

    if(CONFIG_IDF_TARGET_ESP32C3 OR CONFIG_IDF_TARGET_ESP32S3)
        target_link_libraries(${COMPONENT_LIB} PUBLIC btbb)
    endif()
endif()

if(CONFIG_ESP32_PHY_INIT_DATA_IN_PARTITION)
    idf_component_get_property(esp_common_dir esp_common COMPONENT_DIR)
    partition_table_get_partition_info(phy_partition_offset "--partition-type data --partition-subtype phy" "offset")

    if(CONFIG_ESP32_SUPPORT_MULTIPLE_PHY_INIT_DATA_BIN)
        set(phy_init_data_bin "${CMAKE_CURRENT_SOURCE_DIR}/phy_multiple_init_data.bin")
    else()
        set(phy_init_data_bin "${build_dir}/phy_init_data.bin")

        # To get the phy_init_data.bin file, compile phy_init_data.h as a C file and then objcopy
        # the object file to a raw binary
        idf_build_get_property(config_dir CONFIG_DIR)
        add_custom_command(
            OUTPUT ${phy_init_data_bin}
            DEPENDS ${CMAKE_CURRENT_LIST_DIR}/${idf_target}/include/phy_init_data.h
            COMMAND ${CMAKE_C_COMPILER} -x c -c
            -I ${esp_common_dir}/include -I ${CMAKE_CURRENT_LIST_DIR}/include -I ${config_dir}
            -o phy_init_data.obj
            ${CMAKE_CURRENT_LIST_DIR}/${idf_target}/include/phy_init_data.h
            COMMAND ${CMAKE_OBJCOPY} -O binary phy_init_data.obj ${phy_init_data_bin}
            )
        add_custom_target(phy_init_data ALL DEPENDS ${phy_init_data_bin})
        add_dependencies(flash phy_init_data)

        idf_component_get_property(main_args esptool_py FLASH_ARGS)
        idf_component_get_property(sub_args esptool_py FLASH_SUB_ARGS)
    endif()

    set(phy_name "phy")

    esptool_py_flash_target(${phy_name}-flash "${main_args}" "${sub_args}")
    esptool_py_flash_target_image(${phy_name}-flash ${phy_name} "${phy_partition_offset}" "${phy_init_data_bin}")
    esptool_py_flash_target_image(flash ${phy_name} "${phy_partition_offset}" "${phy_init_data_bin}")
endif()
