set(CLICKHOUSE_SERVER_SOURCES
    MetricsTransmitter.cpp
    Server.cpp
)

if (OS_LINUX AND ARCH_AMD64)
    set (LINK_CONFIG_LIB INTERFACE "-Wl,${WHOLE_ARCHIVE} $<TARGET_FILE:clickhouse_server_configs> -Wl,${NO_WHOLE_ARCHIVE}")
endif ()

set (CLICKHOUSE_SERVER_LINK
    PRIVATE
        clickhouse_aggregate_functions
        clickhouse_common_config
        clickhouse_common_io
        clickhouse_common_zookeeper
        clickhouse_dictionaries
        clickhouse_functions
        clickhouse_parsers
        clickhouse_storages_system
        clickhouse_table_functions
        string_utils

    ${LINK_CONFIG_LIB}

    PUBLIC
        daemon
)

clickhouse_program_add(server)

install(FILES config.xml users.xml DESTINATION ${CLICKHOUSE_ETC_DIR}/clickhouse-server COMPONENT clickhouse)

# TODO We actually need this on Mac, FreeBSD.
if (OS_LINUX)
    # Embed default config files as a resource into the binary.
    # This is needed for two purposes:
    # 1. Allow to run the binary without download of any other files.
    # 2. Allow to implement "sudo clickhouse install" tool.

    foreach(CONFIG_FILE config users embedded)
        set(CONFIG_OBJ ${CONFIG_FILE}.o)
        set(CONFIG_OBJS ${CONFIG_OBJS} ${CONFIG_OBJ})

        # https://stackoverflow.com/questions/14776463/compile-and-add-an-object-file-from-a-binary-with-cmake
        add_custom_command(OUTPUT ${CONFIG_OBJ}
            COMMAND cd ${CMAKE_CURRENT_SOURCE_DIR} && ${OBJCOPY_PATH} -I binary ${OBJCOPY_ARCH_OPTIONS} ${CONFIG_FILE}.xml ${CMAKE_CURRENT_BINARY_DIR}/${CONFIG_OBJ}
            COMMAND ${OBJCOPY_PATH} --rename-section .data=.rodata,alloc,load,readonly,data,contents
                ${CMAKE_CURRENT_BINARY_DIR}/${CONFIG_OBJ} ${CMAKE_CURRENT_BINARY_DIR}/${CONFIG_OBJ})

        set_source_files_properties(${CONFIG_OBJ} PROPERTIES EXTERNAL_OBJECT true GENERATED true)
    endforeach(CONFIG_FILE)

    add_library(clickhouse_server_configs STATIC ${CONFIG_OBJS})
    set_target_properties(clickhouse_server_configs PROPERTIES LINKER_LANGUAGE C)

    # whole-archive prevents symbols from being discarded for unknown reason
    # CMake can shuffle each of target_link_libraries arguments with other
    # libraries in linker command. To avoid this we hardcode whole-archive
    # library into single string.
    add_dependencies(clickhouse-server-lib clickhouse_server_configs)
endif ()
