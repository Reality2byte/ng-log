cmake_minimum_required (VERSION 3.22)
project (ng-log
  VERSION 0.9.0
  DESCRIPTION "C++ logging module"
  HOMEPAGE_URL https://github.com/ng-log/ng-log
  LANGUAGES CXX
)

set (CPACK_PACKAGE_NAME ng-log)
set (CPACK_PACKAGE_DESCRIPTION_SUMMARY "C++ logging library")
set (CPACK_PACKAGE_VERSION_MAJOR ${ng-log_VERSION_MAJOR})
set (CPACK_PACKAGE_VERSION_MINOR ${ng-log_VERSION_MINOR})
set (CPACK_PACKAGE_VERSION_PATCH ${ng-log_VERSION_PATCH})
set (CPACK_PACKAGE_VERSION ${ng-log_VERSION})

list (APPEND CMAKE_MODULE_PATH ${ng-log_SOURCE_DIR}/cmake)

include (CheckCXXSourceCompiles)
include (CheckCXXSourceRuns)
include (CheckCXXSymbolExists)
include (CheckIncludeFileCXX)
include (CheckStructHasMember)
include (CheckTypeSize)
include (CMakeDependentOption)
include (CMakePackageConfigHelpers)
include (CMakePushCheckState)
include (CPack)
include (CTest)
include (DetermineGflagsNamespace)
include (GenerateExportHeader)
include (GetCacheVariables)
include (GNUInstallDirs)

option (BUILD_SHARED_LIBS "Build shared libraries" ON)
option (BUILD_EXAMPLES "Build examples" ON)
option (PRINT_UNSYMBOLIZED_STACK_TRACES
  "Print file offsets in traces instead of symbolizing" OFF)
option (WITH_GFLAGS "Use gflags" ON)
option (WITH_GTEST "Use Google Test" ON)
option (WITH_PKGCONFIG "Enable pkg-config support" OFF)
option (WITH_SYMBOLIZE "Enable symbolize module" ON)
option (WITH_TLS "Enable Thread Local Storage (TLS) support" ON)

set (WITH_UNWIND libunwind CACHE STRING "unwind driver")
set_property (CACHE WITH_UNWIND PROPERTY STRINGS none unwind libunwind)

cmake_dependent_option (WITH_GMOCK "Use Google Mock" ON WITH_GTEST OFF)

set (WITH_FUZZING none CACHE STRING "Fuzzing engine")
set_property (CACHE WITH_FUZZING PROPERTY STRINGS none libfuzzer ossfuzz)

if (WITH_UNWIND STREQUAL none)
  set (CMAKE_DISABLE_FIND_PACKAGE_Unwind ON)
endif (WITH_UNWIND STREQUAL none)

if (NOT WITH_GTEST)
  set (CMAKE_DISABLE_FIND_PACKAGE_GTest ON)
endif (NOT WITH_GTEST)

set (CMAKE_C_VISIBILITY_PRESET hidden)
set (CMAKE_CXX_VISIBILITY_PRESET hidden)
set (CMAKE_POSITION_INDEPENDENT_CODE ON)
set (CMAKE_VISIBILITY_INLINES_HIDDEN ON)

set (CMAKE_DEBUG_POSTFIX d)

find_package (GTest 1.11 COMPONENTS GTest OPTIONAL_COMPONENTS GMock NO_MODULE)

if (GTest_FOUND)
  set (HAVE_LIB_GTEST 1)
endif (GTest_FOUND)

if (WITH_GMOCK AND TARGET GTest::gmock)
  set (HAVE_LIB_GMOCK 1)
endif (WITH_GMOCK AND TARGET GTest::gmock)

if (WITH_GFLAGS)
  find_package (gflags 2.2.2)

  if (gflags_FOUND)
    set (HAVE_LIB_GFLAGS 1)
    determine_gflags_namespace (gflags_NAMESPACE)
  endif (gflags_FOUND)
endif (WITH_GFLAGS)

find_package (Threads REQUIRED)
find_package (Unwind)

if (Unwind_FOUND)
  cmake_push_check_state (RESET)
  set (CMAKE_REQUIRED_LIBRARIES unwind::unwind)

  # Check whether linking actually succeeds. ARM toolchains of LLVM unwind
  # implementation do not necessarily provide the _Unwind_Backtrace function
  # which causes the previous check to succeed but the linking to fail.
  check_cxx_symbol_exists (_Unwind_Backtrace unwind.h HAVE__UNWIND_BACKTRACE)
  check_cxx_symbol_exists (_Unwind_GetIP unwind.h HAVE__UNWIND_GETIP)

  check_cxx_symbol_exists (unw_get_reg libunwind.h HAVE_UNW_GET_REG)
  check_cxx_symbol_exists (unw_getcontext libunwind.h HAVE_UNW_GETCONTEXT)
  check_cxx_symbol_exists (unw_init_local libunwind.h HAVE_UNW_INIT_LOCAL)
  check_cxx_symbol_exists (unw_step libunwind.h HAVE_UNW_STEP)

  if (HAVE__UNWIND_BACKTRACE AND HAVE__UNWIND_GETIP)
    set (_HAVE_UNWIND 1)
  endif (HAVE__UNWIND_BACKTRACE AND HAVE__UNWIND_GETIP)

  if (HAVE_UNW_GET_REG AND HAVE_UNW_GETCONTEXT AND HAVE_UNW_INIT_LOCAL AND HAVE_UNW_STEP)
    set (_HAVE_LIBUNWIND 1)
  endif (HAVE_UNW_GET_REG AND HAVE_UNW_GETCONTEXT AND HAVE_UNW_INIT_LOCAL AND HAVE_UNW_STEP)

  if (WITH_UNWIND STREQUAL unwind)
    if (_HAVE_UNWIND)
      set (HAVE_UNWIND 1)
    endif (_HAVE_UNWIND)
  elseif (WITH_UNWIND STREQUAL libunwind)
    if (_HAVE_LIBUNWIND)
      set (HAVE_LIBUNWIND 1)
    endif (_HAVE_LIBUNWIND)
  endif (WITH_UNWIND STREQUAL unwind)

  unset (_HAVE_LIBUNWIND)
  unset (_HAVE_UNWIND)

  cmake_pop_check_state ()
endif (Unwind_FOUND)

check_include_file_cxx (dlfcn.h HAVE_DLFCN_H)
check_include_file_cxx (elf.h HAVE_ELF_H)
check_include_file_cxx (glob.h HAVE_GLOB_H)
check_include_file_cxx (link.h HAVE_LINK_H)
check_include_file_cxx (pwd.h HAVE_PWD_H)
check_include_file_cxx (sys/exec_elf.h HAVE_SYS_EXEC_ELF_H)
check_include_file_cxx (sys/syscall.h HAVE_SYS_SYSCALL_H)
check_include_file_cxx (sys/time.h HAVE_SYS_TIME_H)
check_include_file_cxx (sys/types.h HAVE_SYS_TYPES_H)
check_include_file_cxx (sys/utsname.h HAVE_SYS_UTSNAME_H)
check_include_file_cxx (sys/wait.h HAVE_SYS_WAIT_H)
check_include_file_cxx (syscall.h HAVE_SYSCALL_H)
check_include_file_cxx (syslog.h HAVE_SYSLOG_H)
check_include_file_cxx (ucontext.h HAVE_UCONTEXT_H)
check_include_file_cxx (unistd.h HAVE_UNISTD_H)

check_type_size (mode_t HAVE_MODE_T LANGUAGE CXX)
check_type_size (ssize_t HAVE_SSIZE_T LANGUAGE CXX)

check_cxx_symbol_exists (dladdr dlfcn.h HAVE_DLADDR)
check_cxx_symbol_exists (fcntl fcntl.h HAVE_FCNTL)
check_cxx_symbol_exists (posix_fadvise fcntl.h HAVE_POSIX_FADVISE)
check_cxx_symbol_exists (pread unistd.h HAVE_PREAD)
check_cxx_symbol_exists (pwrite unistd.h HAVE_PWRITE)
check_cxx_symbol_exists (sigaction csignal HAVE_SIGACTION)
check_cxx_symbol_exists (sigaltstack csignal HAVE_SIGALTSTACK)

check_cxx_symbol_exists (backtrace execinfo.h HAVE_EXECINFO_BACKTRACE)
check_cxx_symbol_exists (backtrace_symbols execinfo.h
  HAVE_EXECINFO_BACKTRACE_SYMBOLS)
check_cxx_symbol_exists (_chsize_s io.h HAVE__CHSIZE_S)

cmake_push_check_state (RESET)
set (CMAKE_REQUIRED_LIBRARIES dbghelp)
check_cxx_symbol_exists (UnDecorateSymbolName "windows.h;dbghelp.h" HAVE_DBGHELP)
cmake_pop_check_state ()

if (WITH_FUZZING STREQUAL none)
  # Disable compiler demangler if fuzzing is active; we only want to use the
  # ng-log demangler then.
  check_cxx_symbol_exists (abi::__cxa_demangle cxxabi.h HAVE___CXA_DEMANGLE)
endif (WITH_FUZZING STREQUAL none)

check_cxx_symbol_exists (__argv cstdlib HAVE___ARGV)
check_cxx_symbol_exists (getprogname cstdlib HAVE_GETPROGNAME)
check_cxx_symbol_exists (program_invocation_short_name cerrno HAVE_PROGRAM_INVOCATION_SHORT_NAME)
check_cxx_source_compiles ([=[
#include <cstdlib>
extern char* __progname;
int main() { return __progname != nullptr ? EXIT_SUCCESS : EXIT_FAILURE; }
]=] HAVE___PROGNAME)

if (WITH_TLS)
  set (NGLOG_THREAD_LOCAL_STORAGE 1)
endif (WITH_TLS)

set (_PC_FIELDS
  "uc_mcontext.gregs[REG_PC]"          # Solaris x86 (32 + 64 bit)
  "uc_mcontext.gregs[REG_EIP]"         # Linux (i386)
  "uc_mcontext.gregs[REG_RIP]"         # Linux (x86_64)
  "uc_mcontext.sc_ip"                  # Linux (ia64)
  "uc_mcontext.pc"                     # Linux (mips)
  "uc_mcontext.uc_regs->gregs[PT_NIP]" # Linux (ppc)
  "uc_mcontext.gregs[R15]"             # Linux (arm old [untested])
  "uc_mcontext.arm_pc"                 # Linux (arm arch 5)
  "uc_mcontext.gp_regs[PT_NIP]"        # Suse SLES 11 (ppc64)
  "uc_mcontext.mc_eip"                 # FreeBSD (i386)
  "uc_mcontext.mc_rip"                 # FreeBSD (x86_64 [untested])
  "uc_mcontext.__gregs[_REG_EIP]"      # NetBSD (i386)
  "uc_mcontext.__gregs[_REG_RIP]"      # NetBSD (x86_64)
  "uc_mcontext->ss.eip"                # OS X (i386, <=10.4)
  "uc_mcontext->__ss.__eip"            # OS X (i386, >=10.5)
  "uc_mcontext->ss.rip"                # OS X (x86_64)
  "uc_mcontext->__ss.__rip"            # OS X (>=10.5 [untested])
  "uc_mcontext->ss.srr0"               # OS X (ppc, ppc64 [untested])
  "uc_mcontext->__ss.__srr0"           # OS X (>=10.5 [untested])
)

if (HAVE_UCONTEXT_H AND NOT DEFINED PC_FROM_UCONTEXT)
  cmake_push_check_state (RESET)

  set (CMAKE_REQUIRED_DEFINITIONS -D_GNU_SOURCE)
  set (_PC_HEADERS ucontext.h signal.h)

  foreach (_PC_FIELD IN LISTS _PC_FIELDS)
    foreach (_PC_HEADER IN LISTS _PC_HEADERS)
      # Replace non-alphanumeric characters by underscores since the name will be
      # used as preprocessor definition.
      string (REGEX REPLACE "[^a-zA-Z0-9]" "_" HAVE_UCONTEXT_FIELD_NAME
        "HAVE_PC_FROM_UCONTEXT_${_PC_FIELD}")
      # Strip trailing underscores for readability
      string (REGEX REPLACE "_+$" "" HAVE_UCONTEXT_FIELD_NAME
        "${HAVE_UCONTEXT_FIELD_NAME}")

      check_struct_has_member (ucontext_t ${_PC_FIELD} ${_PC_HEADER}
        ${HAVE_UCONTEXT_FIELD_NAME} LANGUAGE CXX)

      if (${HAVE_UCONTEXT_FIELD_NAME})
        set (PC_FROM_UCONTEXT ${_PC_FIELD} CACHE STRING
          "<${_PC_HEADER}> ucontext_t PC member")
        mark_as_advanced (PC_FROM_UCONTEXT)
        break ()
      endif (${HAVE_UCONTEXT_FIELD_NAME})
    endforeach (_PC_HEADER)

    if (${HAVE_UCONTEXT_FIELD_NAME})
      break ()
    endif (${HAVE_UCONTEXT_FIELD_NAME})
  endforeach (_PC_FIELD)

  cmake_pop_check_state ()
endif (HAVE_UCONTEXT_H AND NOT DEFINED PC_FROM_UCONTEXT)

if (HAVE_EXECINFO_BACKTRACE AND HAVE_EXECINFO_BACKTRACE_SYMBOLS)
  set (HAVE_STACKTRACE 1)
endif (HAVE_EXECINFO_BACKTRACE AND HAVE_EXECINFO_BACKTRACE_SYMBOLS)

if (WITH_SYMBOLIZE)
  if (WIN32 OR CYGWIN)
    cmake_push_check_state (RESET)
    set (CMAKE_REQUIRED_LIBRARIES DbgHelp)

    check_cxx_source_runs ([=[
    #include <windows.h>
    #include <dbghelp.h>
    #include <cstdlib>

    void foobar() { }

    int main()
    {
        HANDLE process = GetCurrentProcess();

        if (!SymInitialize(process, NULL, TRUE))
            return EXIT_FAILURE;

        char buf[sizeof(SYMBOL_INFO) + MAX_SYM_NAME];
        SYMBOL_INFO *symbol = reinterpret_cast<SYMBOL_INFO *>(buf);
        symbol->SizeOfStruct = sizeof(SYMBOL_INFO);
        symbol->MaxNameLen = MAX_SYM_NAME;

        void* const pc = reinterpret_cast<void*>(&foobar);
        BOOL ret = SymFromAddr(process, reinterpret_cast<DWORD64>(pc), 0, symbol);

        return ret ? EXIT_SUCCESS : EXIT_FAILURE;
    }
    ]=] HAVE_SYMBOLIZE)

    cmake_pop_check_state ()

    if (HAVE_SYMBOLIZE)
      set (HAVE_STACKTRACE 1)
    endif (HAVE_SYMBOLIZE)
  elseif (APPLE AND HAVE_DLADDR)
    set (HAVE_SYMBOLIZE 1)
  elseif (UNIX)
    if (HAVE_ELF_H OR HAVE_SYS_EXEC_ELF_H)
      set (HAVE_SYMBOLIZE 1)
    endif (HAVE_ELF_H OR HAVE_SYS_EXEC_ELF_H)
  endif (WIN32 OR CYGWIN)
endif (WITH_SYMBOLIZE)

# CMake manages symbolize availability. The definition is necessary only when
# building the library.
add_compile_definitions (NGLOG_NO_SYMBOLIZE_DETECTION)

check_cxx_symbol_exists (gmtime_r "cstdlib;ctime" HAVE_GMTIME_R)
check_cxx_symbol_exists (localtime_r "cstdlib;ctime" HAVE_LOCALTIME_R)

set (SIZEOF_VOID_P ${CMAKE_SIZEOF_VOID_P})

# fopen/open on Cygwin can not handle unix-type paths like /home/....
# therefore we translate TEST_SRC_DIR to windows-path.
if (CYGWIN)
  execute_process (COMMAND cygpath.exe -m ${ng-log_SOURCE_DIR}
                   OUTPUT_STRIP_TRAILING_WHITESPACE
                   OUTPUT_VARIABLE TEST_SRC_DIR)
  set (TEST_SRC_DIR \"${TEST_SRC_DIR}\")
else (CYGWIN)
  set (TEST_SRC_DIR \"${ng-log_SOURCE_DIR}\")
endif (CYGWIN)

configure_file (src/config.h.cmake.in config.h)

set (_ng-log_CMake_BINDIR ${CMAKE_INSTALL_BINDIR})
set (_ng-log_CMake_INCLUDE_DIR ${CMAKE_INSTALL_INCLUDEDIR})
set (_ng-log_CMake_LIBDIR ${CMAKE_INSTALL_LIBDIR})
set (_ng-log_CMake_INSTALLDIR ${_ng-log_CMake_LIBDIR}/cmake/ng-log)

set (_ng-log_CMake_DIR ng-log/cmake)
set (_ng-log_CMake_DATADIR ${CMAKE_INSTALL_DATAROOTDIR}/${_ng-log_CMake_DIR})
set (_ng-log_BINARY_CMake_DATADIR
  ${ng-log_BINARY_DIR}/${_ng-log_CMake_DATADIR})

# Add additional CMake find modules here.
set (_ng-log_CMake_MODULES)

if (Unwind_FOUND)
  # Copy the module only if libunwind is actually used.
  list (APPEND _ng-log_CMake_MODULES ${ng-log_SOURCE_DIR}/cmake/FindUnwind.cmake)
endif (Unwind_FOUND)

# Generate file name for each module in the binary directory
foreach (_file ${_ng-log_CMake_MODULES})
  get_filename_component (_module "${_file}" NAME)

  list (APPEND _ng-log_BINARY_CMake_MODULES
    ${_ng-log_BINARY_CMake_DATADIR}/${_module})
endforeach (_file)

if (_ng-log_CMake_MODULES)
  # Copy modules to binary directory during the build
  add_custom_command (OUTPUT ${_ng-log_BINARY_CMake_MODULES}
    COMMAND ${CMAKE_COMMAND} -E make_directory
    ${_ng-log_BINARY_CMake_DATADIR}
    COMMAND ${CMAKE_COMMAND} -E copy ${_ng-log_CMake_MODULES}
    ${_ng-log_BINARY_CMake_DATADIR}
    DEPENDS ${_ng-log_CMake_MODULES}
    COMMENT "Copying find modules..."
  )
endif (_ng-log_CMake_MODULES)

set (NGLOG_PUBLIC_H
  ${ng-log_BINARY_DIR}/ng-log/export.h
  src/ng-log/log_severity.h
  src/ng-log/logging.h
  src/ng-log/platform.h
  src/ng-log/raw_logging.h
  src/ng-log/stl_logging.h
  src/ng-log/types.h
  src/ng-log/flags.h
  src/ng-log/vlog_is_on.h
)

set (NGLOG_SRCS
  ${NGLOG_PUBLIC_H}
  src/base/commandlineflags.h
  src/demangle.cc
  src/demangle.h
  src/flags.cc
  src/initializer.h
  src/logging.cc
  src/raw_logging.cc
  src/signalhandler.cc
  src/stacktrace.cc
  src/stacktrace.h
  src/symbolize.cc
  src/symbolize.h
  src/utilities.cc
  src/utilities.h
  src/vlog_is_on.cc
)

# NOTE MSYS2 defines both WIN32 and UNIX. Do not use windows port in this case.
if ((CYGWIN OR WIN32) AND NOT UNIX)
  list (APPEND NGLOG_SRCS
    src/windows/port.cc
    src/windows/port.h
  )
  set (_ng-log_USE_WINDOWS_PORT TRUE)
endif ((CYGWIN OR WIN32) AND NOT UNIX)

add_library (ng-log_internal OBJECT
  ${_ng-log_BINARY_CMake_MODULES}
  ${NGLOG_SRCS}
)
target_compile_features (ng-log_internal PUBLIC $<TARGET_PROPERTY:ng-log,COMPILE_FEATURES>)
set_target_properties (ng-log_internal PROPERTIES DEFINE_SYMBOL ng_log_EXPORTS)

# Some generators (such as Xcode) do not generate any output if the target does
# not reference at least one source file.
set (_ng-log_EMPTY_SOURCE ${ng-log_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/ng-log.cc)

add_custom_command (
  OUTPUT ${_ng-log_EMPTY_SOURCE}
  COMMAND ${CMAKE_COMMAND} -E touch ${_ng-log_EMPTY_SOURCE}
)

add_library (ng-log
  $<TARGET_OBJECTS:ng-log_internal>
  ${_ng-log_EMPTY_SOURCE}
)
target_compile_features (ng-log PUBLIC cxx_std_14)

add_library (ng-log::ng-log ALIAS ng-log)

set (ng-log_libraries_options_for_static_linking)

# CMake always uses the generated export header
target_compile_definitions (ng-log PUBLIC NGLOG_USE_EXPORT)

if (_ng-log_USE_WINDOWS_PORT)
  target_compile_definitions (ng-log PRIVATE NGLOG_USE_WINDOWS_PORT)
endif (_ng-log_USE_WINDOWS_PORT)

unset (_ng-log_USE_WINDOWS_PORT)

if (WIN32)
  # Do not define min and max as macros
  target_compile_definitions (ng-log PRIVATE NOMINMAX)
  # Exclude unnecessary funcitonality
  target_compile_definitions (ng-log PRIVATE WIN32_LEAN_AND_MEAN)
endif (WIN32)

if (HAVE_LIB_GFLAGS)
  target_compile_definitions (ng-log PUBLIC NGLOG_USE_GFLAGS)
endif (HAVE_LIB_GFLAGS)

if (Unwind_FOUND)
  target_link_libraries (ng-log PRIVATE unwind::unwind)
  set (ng-log_libraries_options_for_static_linking "${ng-log_libraries_options_for_static_linking} -lunwind")
  set (Unwind_DEPENDENCY "find_dependency (Unwind ${Unwind_VERSION})")
endif (Unwind_FOUND)

if (HAVE_DBGHELP)
  target_link_libraries (ng-log PRIVATE dbghelp)
  set (ng-log_libraries_options_for_static_linking "${ng-log_libraries_options_for_static_linking} -ldbghelp")
endif (HAVE_DBGHELP)

target_link_libraries (ng-log PRIVATE Threads::Threads)

if (CMAKE_THREAD_LIBS_INIT)
  set (ng-log_libraries_options_for_static_linking "${ng-log_libraries_options_for_static_linking} ${CMAKE_THREAD_LIBS_INIT}")
endif (CMAKE_THREAD_LIBS_INIT)

if (gflags_FOUND)
  # Prefer the gflags target that uses double colon convention
  if (TARGET gflags::gflags)
    target_link_libraries (ng-log PUBLIC gflags::gflags)
  else (TARGET gflags::gflags)
    target_link_libraries (ng-log PUBLIC gflags)
  endif (TARGET gflags::gflags)

  set (ng-log_libraries_options_for_static_linking "${ng-log_libraries_options_for_static_linking} -lgflags")
endif (gflags_FOUND)

if (ANDROID)
  target_link_libraries (ng-log PRIVATE log)
  set (ng-log_libraries_options_for_static_linking "${ng-log_libraries_options_for_static_linking} -llog")
endif (ANDROID)

set_target_properties (ng-log PROPERTIES VERSION ${ng-log_VERSION})
set_target_properties (ng-log PROPERTIES SOVERSION 0)

if (CYGWIN OR WIN32)
  target_compile_definitions (ng-log PUBLIC NGLOG_NO_ABBREVIATED_SEVERITIES)
endif (CYGWIN OR WIN32)

set_target_properties (ng-log PROPERTIES PUBLIC_HEADER "${NGLOG_PUBLIC_H}")

target_include_directories (ng-log BEFORE PUBLIC
  "$<BUILD_INTERFACE:${ng-log_BINARY_DIR}>"
  "$<BUILD_INTERFACE:${ng-log_SOURCE_DIR}/src>"
  "$<INSTALL_INTERFACE:${_ng-log_CMake_INCLUDE_DIR}>"
  PRIVATE ${ng-log_BINARY_DIR}
  PRIVATE ${ng-log_SOURCE_DIR}/src)

if (CYGWIN OR WIN32)
  target_include_directories (ng-log_internal PUBLIC
    "$<BUILD_INTERFACE:${ng-log_SOURCE_DIR}/src/windows>"
    PRIVATE ${ng-log_SOURCE_DIR}/src/windows)

  target_include_directories (ng-log PUBLIC
    "$<BUILD_INTERFACE:${ng-log_SOURCE_DIR}/src/windows>"
    PRIVATE ${ng-log_SOURCE_DIR}/src/windows)
endif (CYGWIN OR WIN32)

set_target_properties (ng-log PROPERTIES DEFINE_SYMBOL ng_log_EXPORTS)

target_include_directories (ng-log_internal PUBLIC
  $<TARGET_PROPERTY:ng-log,INCLUDE_DIRECTORIES>)
target_compile_definitions (ng-log_internal PUBLIC
  $<TARGET_PROPERTY:ng-log,COMPILE_DEFINITIONS>
  PRIVATE ng_log_EXPORTS)

generate_export_header (ng-log
  BASE_NAME NGLOG
  EXPORT_FILE_NAME ${ng-log_BINARY_DIR}/ng-log/export.h)

string (STRIP "${ng-log_libraries_options_for_static_linking}" ng-log_libraries_options_for_static_linking)

if (WITH_PKGCONFIG)
  set (VERSION ${ng-log_VERSION})
  set (prefix ${CMAKE_INSTALL_PREFIX})
  set (exec_prefix ${CMAKE_INSTALL_FULL_BINDIR})
  set (libdir ${CMAKE_INSTALL_FULL_LIBDIR})
  set (includedir ${CMAKE_INSTALL_FULL_INCLUDEDIR})

  configure_file (
    "${ng-log_SOURCE_DIR}/libng-log.pc.in"
    "${ng-log_BINARY_DIR}/libng-log.pc"
    @ONLY
  )

  unset (VERSION)
  unset (prefix)
  unset (exec_prefix)
  unset (libdir)
  unset (includedir)
endif (WITH_PKGCONFIG)

# Unit testing

if (NOT WITH_FUZZING STREQUAL "none")
  add_executable (fuzz_demangle
    src/fuzz_demangle.cc
  )

  if (WITH_FUZZING STREQUAL "ossfuzz")
    set (LIB_FUZZING_ENGINE $ENV{LIB_FUZZING_ENGINE})
    target_link_libraries (fuzz_demangle PRIVATE ng-log ${LIB_FUZZING_ENGINE})
  elseif (WITH_FUZZING STREQUAL "libfuzzer")
    target_compile_options (fuzz_demangle PRIVATE -fsanitize=fuzzer)
    target_link_libraries (fuzz_demangle PRIVATE ng-log)
  else (WITH_FUZZING STREQUAL "libfuzzer")
    message (FATAL_ERROR "Unsupported fuzzing engine ${WITH_FUZZING}")
  endif (WITH_FUZZING STREQUAL "ossfuzz")
endif (NOT WITH_FUZZING STREQUAL "none")

if (BUILD_TESTING)
  add_library (ng-log_test INTERFACE)
  target_link_libraries (ng-log_test INTERFACE $<TARGET_OBJECTS:ng-log_internal>
    $<TARGET_PROPERTY:ng-log,LINK_LIBRARIES>)
  target_compile_definitions (ng-log_test INTERFACE NGLOG_STATIC_DEFINE
    $<TARGET_PROPERTY:ng-log,COMPILE_DEFINITIONS>)
  target_compile_features (ng-log_test INTERFACE
    $<TARGET_PROPERTY:ng-log,COMPILE_FEATURES>)
  target_include_directories (ng-log_test INTERFACE $<TARGET_PROPERTY:ng-log,INCLUDE_DIRECTORIES>)

  if (HAVE_LIB_GTEST)
    target_link_libraries (ng-log_test INTERFACE GTest::gtest)
  endif (HAVE_LIB_GTEST)

  if (HAVE_LIB_GMOCK)
    target_link_libraries (ng-log_test INTERFACE GTest::gmock)
  endif (HAVE_LIB_GMOCK)

  add_executable (logging_unittest
    src/logging_unittest.cc
  )

  target_link_libraries (logging_unittest PRIVATE ng-log_test)

  add_executable (stl_logging_unittest
    src/stl_logging_unittest.cc
  )

  target_link_libraries (stl_logging_unittest PRIVATE ng-log_test)

  if (HAVE_SYMBOLIZE)
    add_executable (symbolize_unittest
      src/symbolize_unittest.cc
    )

    target_link_libraries (symbolize_unittest PRIVATE ng-log_test)
  endif (HAVE_SYMBOLIZE)

  add_executable (demangle_unittest
    src/demangle_unittest.cc
  )

  target_link_libraries (demangle_unittest PRIVATE ng-log_test)

  add_test (NAME demangle COMMAND demangle_unittest)

  if (HAVE___CXA_DEMANGLE)
    # Demangle tests use a different (reduced) representation of symbols
    set_tests_properties (demangle PROPERTIES DISABLED ON)
  endif (HAVE___CXA_DEMANGLE)

  if (HAVE_STACKTRACE)
    add_executable (stacktrace_unittest
      src/stacktrace_unittest.cc
    )

    target_link_libraries (stacktrace_unittest PRIVATE ng-log_test)
  endif (HAVE_STACKTRACE)

  add_executable (utilities_unittest
    src/utilities_unittest.cc
  )

  target_link_libraries (utilities_unittest PRIVATE ng-log_test)

  if (HAVE_STACKTRACE AND HAVE_SYMBOLIZE)
    add_executable (signalhandler_unittest
      src/signalhandler_unittest.cc
    )

    target_link_libraries (signalhandler_unittest PRIVATE ng-log_test)
  endif (HAVE_STACKTRACE AND HAVE_SYMBOLIZE)

  add_test (NAME logging COMMAND logging_unittest)

  set_tests_properties (logging PROPERTIES TIMEOUT 30)
  # MacOS diff is not deterministic: use the output to determine whether the
  # test passed.
  set_tests_properties (logging PROPERTIES PASS_REGULAR_EXPRESSION ".*\nPASS\n.*")

  # FIXME: Skip flaky test
  set_tests_properties (logging PROPERTIES SKIP_REGULAR_EXPRESSION
    "Check failed: time_ns within LogTimes::LOG_PERIOD_TOL_NS of LogTimes::LOG_PERIOD_NS")

  if (APPLE)
    # FIXME: Skip flaky test
    set_property (TEST logging APPEND PROPERTY SKIP_REGULAR_EXPRESSION
      "unexpected new.*PASS\nTest with golden file failed. We'll try to show the diff:")
  endif (APPLE)

  if (TARGET signalhandler_unittest)
    add_test (NAME signalhandler COMMAND signalhandler_unittest)
  endif (TARGET signalhandler_unittest)

  if (TARGET stacktrace_unittest)
    add_test (NAME stacktrace COMMAND stacktrace_unittest)
    set_tests_properties (stacktrace PROPERTIES TIMEOUT 30)

    if (APPLE)
      set_tests_properties (stacktrace PROPERTIES SKIP_REGULAR_EXPRESSION
        [=[\(&expected_range[[]5[]]\)->start < \(&expected_range[[]5[]]\)->end \(0x1 vs\. 0x1\)]=]
      )
    endif (APPLE)
  endif (TARGET stacktrace_unittest)

  add_test (NAME stl_logging COMMAND stl_logging_unittest)

  if (TARGET symbolize_unittest)
    add_test (NAME symbolize COMMAND symbolize_unittest)

    # FIXME: Skip flaky test when compiled in C++20 mode
    set_tests_properties (symbolize PROPERTIES SKIP_REGULAR_EXPRESSION
      [=[Check failed: streq\("nonstatic_func"\, TrySymbolize\(\(void \*\)\(&nonstatic_func\)\)\)]=])
  endif (TARGET symbolize_unittest)

  if (HAVE_LIB_GMOCK)
    add_executable (mock-log_unittest
      src/mock-log_unittest.cc
      src/mock-log.h
    )

    target_link_libraries (mock-log_unittest PRIVATE ng-log_test)

    add_test (NAME mock-log COMMAND mock-log_unittest)
  endif (HAVE_LIB_GMOCK)

  # Generate an initial cache

  get_cache_variables (_CACHEVARS)

  set (_INITIAL_CACHE
    ${ng-log_BINARY_DIR}/test_package_config/ng-log_package_config_initial_cache.cmake)

  # Package config test

  add_test (NAME cmake_package_config_init COMMAND ${CMAKE_COMMAND}
    -DTEST_BINARY_DIR=${ng-log_BINARY_DIR}/test_package_config
    -DINITIAL_CACHE=${_INITIAL_CACHE}
    -DCACHEVARS=${_CACHEVARS}
    -P ${ng-log_SOURCE_DIR}/cmake/TestInitPackageConfig.cmake
  )

  add_test (NAME cmake_package_config_generate COMMAND ${CMAKE_COMMAND}
    -DGENERATOR=${CMAKE_GENERATOR}
    -DGENERATOR_PLATFORM=${CMAKE_GENERATOR_PLATFORM}
    -DGENERATOR_TOOLSET=${CMAKE_GENERATOR_TOOLSET}
    -DINITIAL_CACHE=${_INITIAL_CACHE}
    -DPACKAGE_DIR=${ng-log_BINARY_DIR}
    -DPATH=$ENV{PATH}
    -DSOURCE_DIR=${ng-log_SOURCE_DIR}/src/package_config_unittest/working_config
    -DTEST_BINARY_DIR=${ng-log_BINARY_DIR}/test_package_config/working_config
    -P ${ng-log_SOURCE_DIR}/cmake/TestPackageConfig.cmake
  )

  add_test (NAME cmake_package_config_build COMMAND
    ${CMAKE_COMMAND} --build ${ng-log_BINARY_DIR}/test_package_config/working_config
                     --config $<CONFIG>
  )

  add_test (NAME cmake_package_config_cleanup COMMAND ${CMAKE_COMMAND} -E
    remove_directory
    ${ng-log_BINARY_DIR}/test_package_config
  )

  # Fixtures setup
  set_tests_properties (cmake_package_config_init PROPERTIES FIXTURES_SETUP
    cmake_package_config)
  set_tests_properties (cmake_package_config_generate PROPERTIES FIXTURES_SETUP
    cmake_package_config_working)

  # Fixtures cleanup
  set_tests_properties (cmake_package_config_cleanup PROPERTIES FIXTURES_CLEANUP
    cmake_package_config)

  # Fixture requirements
  set_tests_properties (cmake_package_config_generate PROPERTIES
    FIXTURES_REQUIRED cmake_package_config)
  set_tests_properties (cmake_package_config_build PROPERTIES
    FIXTURES_REQUIRED "cmake_package_config;cmake_package_config_working")

  add_executable (cleanup_immediately_unittest
    src/cleanup_immediately_unittest.cc)

  target_link_libraries (cleanup_immediately_unittest PRIVATE ng-log_test)

  add_executable (cleanup_with_absolute_prefix_unittest
    src/cleanup_with_absolute_prefix_unittest.cc)

  target_link_libraries (cleanup_with_absolute_prefix_unittest PRIVATE ng-log_test)

  add_executable (cleanup_with_relative_prefix_unittest
    src/cleanup_with_relative_prefix_unittest.cc)

  target_link_libraries (cleanup_with_relative_prefix_unittest PRIVATE ng-log_test)

  set (CLEANUP_LOG_DIR ${ng-log_BINARY_DIR}/cleanup_tests)

  add_test (NAME cleanup_init COMMAND
    ${CMAKE_COMMAND} -E make_directory ${CLEANUP_LOG_DIR})
  add_test (NAME cleanup_logdir COMMAND
    ${CMAKE_COMMAND} -E remove_directory ${CLEANUP_LOG_DIR})
  add_test (NAME cleanup_immediately COMMAND
    ${CMAKE_COMMAND}
    -DLOGCLEANUP=$<TARGET_FILE:cleanup_immediately_unittest>
    # NOTE The trailing slash is important
    -DTEST_DIR=${CLEANUP_LOG_DIR}/
    -P ${ng-log_SOURCE_DIR}/cmake/RunCleanerTest1.cmake
    WORKING_DIRECTORY ${ng-log_BINARY_DIR})
  add_test (NAME cleanup_with_absolute_prefix COMMAND
    ${CMAKE_COMMAND}
    -DLOGCLEANUP=$<TARGET_FILE:cleanup_with_absolute_prefix_unittest>
    -DTEST_DIR=${ng-log_BINARY_DIR}/
    -P ${ng-log_SOURCE_DIR}/cmake/RunCleanerTest2.cmake
    WORKING_DIRECTORY ${ng-log_BINARY_DIR})
  add_test (NAME cleanup_with_relative_prefix COMMAND
    ${CMAKE_COMMAND}
    -DLOGCLEANUP=$<TARGET_FILE:cleanup_with_relative_prefix_unittest>
    -DTEST_DIR=${ng-log_BINARY_DIR}/
    -DTEST_SUBDIR=test_subdir/
    -P ${ng-log_SOURCE_DIR}/cmake/RunCleanerTest3.cmake
    WORKING_DIRECTORY ${ng-log_BINARY_DIR})

  # Fixtures setup
  set_tests_properties (cleanup_init PROPERTIES FIXTURES_SETUP logcleanuptest)
  ## Fixtures cleanup
  set_tests_properties (cleanup_logdir PROPERTIES FIXTURES_CLEANUP logcleanuptest)
  # Fixture requirements
  set_tests_properties (cleanup_immediately PROPERTIES FIXTURES_REQUIRED logcleanuptest)
  set_tests_properties (cleanup_with_absolute_prefix PROPERTIES FIXTURES_REQUIRED logcleanuptest)
  set_tests_properties (cleanup_with_relative_prefix PROPERTIES FIXTURES_REQUIRED logcleanuptest)

  add_executable (striplog0_unittest
    src/striplog_unittest.cc
  )
  target_compile_definitions (striplog0_unittest PRIVATE NGLOG_STRIP_LOG=0)
  target_link_libraries (striplog0_unittest PRIVATE ng-log_test)

  add_test (NAME striplog0 COMMAND striplog0_unittest)

  add_executable (striplog2_unittest
    src/striplog_unittest.cc
  )
  target_compile_definitions (striplog2_unittest PRIVATE NGLOG_STRIP_LOG=2)
  target_link_libraries (striplog2_unittest PRIVATE ng-log_test)

  add_test (NAME striplog2 COMMAND striplog2_unittest)

  add_executable (striplog10_unittest
    src/striplog_unittest.cc
  )
  target_compile_definitions (striplog10_unittest PRIVATE NGLOG_STRIP_LOG=10)
  target_link_libraries (striplog10_unittest PRIVATE ng-log_test)

  add_test (NAME striplog10 COMMAND striplog10_unittest)

  set_tests_properties (
    striplog0
    striplog2
    striplog10
    PROPERTIES WILL_FAIL ON
  )

  add_test (NAME log_severity_constants COMMAND ${CMAKE_CTEST_COMMAND}
    --build-config $<CONFIG>
    --build-and-test
    "${ng-log_SOURCE_DIR}/src/log_severity_unittest"
    "${ng-log_BINARY_DIR}/Tests/log_severity_constants"
    --build-generator ${CMAKE_GENERATOR}
    --build-makeprogram ${CMAKE_MAKE_PROGRAM}
    --build-target ng-log_log_severity_constants
    --build-options
    -DCMAKE_BUILD_TYPE=$<CONFIG>
    -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
    -DCMAKE_GENERATOR_PLATFORM=${CMAKE_GENERATOR_PLATFORM}
    -DCMAKE_GENERATOR_TOOLSET=${CMAKE_GENERATOR_TOOLSET}
    -Dng-log_DIR=${ng-log_BINARY_DIR}
  )
  set_tests_properties (log_severity_constants PROPERTIES
    PASS_REGULAR_EXPRESSION "NGLOG_COMPACT_LOG_[1-3]"
  )

  add_test (NAME log_severity_conversion COMMAND ${CMAKE_CTEST_COMMAND}
    --build-config $<CONFIG>
    --build-and-test
    "${ng-log_SOURCE_DIR}/src/log_severity_unittest"
    "${ng-log_BINARY_DIR}/Tests/log_severity_conversion"
    --build-generator ${CMAKE_GENERATOR}
    --build-makeprogram ${CMAKE_MAKE_PROGRAM}
    --build-target ng-log_log_severity_conversion
    --build-options
    -DCMAKE_BUILD_TYPE=$<CONFIG>
    -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
    -DCMAKE_GENERATOR_PLATFORM=${CMAKE_GENERATOR_PLATFORM}
    -DCMAKE_GENERATOR_TOOLSET=${CMAKE_GENERATOR_TOOLSET}
    -Dng-log_DIR=${ng-log_BINARY_DIR}
  )

  if (CMAKE_COMPILER_IS_GNUCXX)
    set_tests_properties (log_severity_conversion PROPERTIES
      PASS_REGULAR_EXPRESSION "error: invalid conversion from (‘|')int(’|')"
    )
  elseif (CMAKE_CXX_COMPILER_ID MATCHES Clang)
    set_tests_properties (log_severity_conversion PROPERTIES
      PASS_REGULAR_EXPRESSION "no known conversion from 'int'"
    )
  elseif (MSVC)
    set_tests_properties (log_severity_conversion PROPERTIES
      PASS_REGULAR_EXPRESSION "error C2440"
    )
  else (CMAKE_COMPILER_IS_GNUCXX)
    message (AUTHOR_WARNING
      "Unsupported C++ compiler ${CMAKE_CXX_COMPILER_ID}: "
      "log_severity_conversion test will be disabled"
    )
    set_tests_properties (log_severity_conversion DISABLED ON)
  endif (CMAKE_COMPILER_IS_GNUCXX)

  add_test (NAME includes_logging COMMAND ${CMAKE_CTEST_COMMAND}
    --build-config $<CONFIG>
    --build-and-test
    "${ng-log_SOURCE_DIR}/src/includes_unittest"
    "${ng-log_BINARY_DIR}/Tests/includes_logging"
    --build-generator ${CMAKE_GENERATOR}
    --build-makeprogram ${CMAKE_MAKE_PROGRAM}
    --build-target ng-log_includes_logging
    --build-options
    -DCMAKE_BUILD_TYPE=$<CONFIG>
    -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
    -DCMAKE_GENERATOR_PLATFORM=${CMAKE_GENERATOR_PLATFORM}
    -DCMAKE_GENERATOR_TOOLSET=${CMAKE_GENERATOR_TOOLSET}
    -Dng-log_DIR=${ng-log_BINARY_DIR}
  )

  add_test (NAME includes_vlog_is_on COMMAND ${CMAKE_CTEST_COMMAND}
    --build-config $<CONFIG>
    --build-and-test
    "${ng-log_SOURCE_DIR}/src/includes_unittest"
    "${ng-log_BINARY_DIR}/Tests/includes_vlog_is_on"
    --build-generator ${CMAKE_GENERATOR}
    --build-makeprogram ${CMAKE_MAKE_PROGRAM}
    --build-target ng-log_includes_vlog_is_on
    --build-options
    -DCMAKE_BUILD_TYPE=$<CONFIG>
    -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
    -DCMAKE_GENERATOR_PLATFORM=${CMAKE_GENERATOR_PLATFORM}
    -DCMAKE_GENERATOR_TOOLSET=${CMAKE_GENERATOR_TOOLSET}
    -Dng-log_DIR=${ng-log_BINARY_DIR}
  )

  add_test (NAME includes_raw_logging COMMAND ${CMAKE_CTEST_COMMAND}
    --build-config $<CONFIG>
    --build-and-test
    "${ng-log_SOURCE_DIR}/src/includes_unittest"
    "${ng-log_BINARY_DIR}/Tests/includes_raw_logging"
    --build-generator ${CMAKE_GENERATOR}
    --build-makeprogram ${CMAKE_MAKE_PROGRAM}
    --build-target ng-log_includes_raw_logging
    --build-options
    -DCMAKE_BUILD_TYPE=$<CONFIG>
    -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
    -DCMAKE_GENERATOR_PLATFORM=${CMAKE_GENERATOR_PLATFORM}
    -DCMAKE_GENERATOR_TOOLSET=${CMAKE_GENERATOR_TOOLSET}
    -Dng-log_DIR=${ng-log_BINARY_DIR}
  )

  add_test (NAME includes_stl_logging COMMAND ${CMAKE_CTEST_COMMAND}
    --build-config $<CONFIG>
    --build-and-test
    "${ng-log_SOURCE_DIR}/src/includes_unittest"
    "${ng-log_BINARY_DIR}/Tests/includes_stl_logging"
    --build-generator ${CMAKE_GENERATOR}
    --build-makeprogram ${CMAKE_MAKE_PROGRAM}
    --build-target ng-log_includes_stl_logging
    --build-options
    -DCMAKE_BUILD_TYPE=$<CONFIG>
    -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
    -DCMAKE_GENERATOR_PLATFORM=${CMAKE_GENERATOR_PLATFORM}
    -DCMAKE_GENERATOR_TOOLSET=${CMAKE_GENERATOR_TOOLSET}
    -Dng-log_DIR=${ng-log_BINARY_DIR}
  )

  add_test (NAME dcheck_on COMMAND ${CMAKE_CTEST_COMMAND}
    --build-config Debug
    --build-and-test
    "${ng-log_SOURCE_DIR}/src/dcheck_unittest"
    "${ng-log_BINARY_DIR}/Tests/dcheck_on"
    --build-generator ${CMAKE_GENERATOR}
    --build-generator-platform "${CMAKE_GENERATOR_PLATFORM}"
    --build-generator-toolset "${CMAKE_GENERATOR_TOOLSET}"
    --build-makeprogram ${CMAKE_MAKE_PROGRAM}
    --build-target ng-log_dcheck
    --build-options
    -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
    -DCMAKE_FIND_PACKAGE_NO_PACKAGE_REGISTRY=ON
    -Dng-log_DIR=${ng-log_BINARY_DIR}
    --test-command ng-log_dcheck
  )
  set_tests_properties (dcheck_on PROPERTIES
    DISABLED $<NOT:$<CONFIG:Debug,RelWithDebInfo>>
    ENVIRONMENT_MODIFICATION "PATH=path_list_prepend:$<TARGET_FILE_DIR:ng-log>"
    PASS_REGULAR_EXPRESSION "Assert failed: false"
  )

  add_test (NAME dcheck_off COMMAND ${CMAKE_CTEST_COMMAND}
    --build-config Release
    --build-and-test
    "${ng-log_SOURCE_DIR}/src/dcheck_unittest"
    "${ng-log_BINARY_DIR}/Tests/dcheck_off"
    --build-generator ${CMAKE_GENERATOR}
    --build-generator-platform "${CMAKE_GENERATOR_PLATFORM}"
    --build-generator-toolset "${CMAKE_GENERATOR_TOOLSET}"
    --build-makeprogram ${CMAKE_MAKE_PROGRAM}
    --build-target ng-log_dcheck
    --build-options
    -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
    -DCMAKE_FIND_PACKAGE_NO_PACKAGE_REGISTRY=ON
    -Dng-log_DIR=${ng-log_BINARY_DIR}
    --test-command ng-log_dcheck
  )
  # There should be no output
  set_tests_properties (dcheck_off PROPERTIES
    DISABLED $<NOT:$<CONFIG:Release,MinSizeRel>>
    ENVIRONMENT_MODIFICATION "PATH=path_list_prepend:$<TARGET_FILE_DIR:ng-log>"
    PASS_REGULAR_EXPRESSION ""
  )
endif (BUILD_TESTING)

if (BUILD_EXAMPLES)
  add_executable (custom_sink_example examples/custom_sink.cc)
  target_link_libraries (custom_sink_example PRIVATE ng-log::ng-log)
endif (BUILD_EXAMPLES)

install (TARGETS ng-log
  EXPORT ng-log-targets
  RUNTIME DESTINATION ${_ng-log_CMake_BINDIR}
  PUBLIC_HEADER DESTINATION ${_ng-log_CMake_INCLUDE_DIR}/ng-log
  LIBRARY DESTINATION ${_ng-log_CMake_LIBDIR}
  ARCHIVE DESTINATION ${_ng-log_CMake_LIBDIR})

if (WITH_PKGCONFIG)
  install (
    FILES "${ng-log_BINARY_DIR}/libng-log.pc"
    DESTINATION "${_ng-log_CMake_LIBDIR}/pkgconfig"
  )
endif (WITH_PKGCONFIG)

set (ng-log_CMake_VERSION 3.0)

if (gflags_FOUND)
  # Ensure clients locate only the package config and not third party find
  # modules having the same name. This avoid cmake_policy PUSH/POP errors.
  if (CMAKE_VERSION VERSION_LESS 3.9)
    set (gflags_DEPENDENCY "find_dependency (gflags ${gflags_VERSION})")
  else (CMAKE_VERSION VERSION_LESS 3.9)
    # Passing additional find_package arguments to find_dependency is possible
    # starting with CMake 3.9.
    set (ng-log_CMake_VERSION 3.9)
    set (gflags_DEPENDENCY "find_dependency (gflags ${gflags_VERSION} NO_MODULE)")
  endif (CMAKE_VERSION VERSION_LESS 3.9)
endif (gflags_FOUND)

configure_package_config_file (ng-log-config.cmake.in
  ${ng-log_BINARY_DIR}/ng-log-config.cmake
  INSTALL_DESTINATION ${_ng-log_CMake_INSTALLDIR}
  NO_CHECK_REQUIRED_COMPONENTS_MACRO)

write_basic_package_version_file (
  ${ng-log_BINARY_DIR}/ng-log-config-version.cmake
  COMPATIBILITY SameMajorVersion)

export (TARGETS ng-log NAMESPACE ng-log:: FILE ng-log-targets.cmake)
export (PACKAGE ng-log)

get_filename_component (_PREFIX "${CMAKE_INSTALL_PREFIX}" ABSOLUTE)

# Directory containing the find modules relative to the config install
# directory.
file (RELATIVE_PATH ng-log_REL_CMake_MODULES
  ${_PREFIX}/${_ng-log_CMake_INSTALLDIR}
  ${_PREFIX}/${_ng-log_CMake_DATADIR}/ng-log-modules.cmake)

get_filename_component (ng-log_REL_CMake_DATADIR ${ng-log_REL_CMake_MODULES}
  DIRECTORY)

set (ng-log_FULL_CMake_DATADIR
  ${ng-log_BINARY_DIR}/${_ng-log_CMake_DATADIR})

configure_file (ng-log-modules.cmake.in
  ${ng-log_BINARY_DIR}/ng-log-modules.cmake @ONLY)

install (CODE
"
set (ng-log_FULL_CMake_DATADIR \"\\\${CMAKE_CURRENT_LIST_DIR}/${ng-log_REL_CMake_DATADIR}\")
set (ng-log_DATADIR_DESTINATION ${_ng-log_CMake_INSTALLDIR})

if (NOT IS_ABSOLUTE ${_ng-log_CMake_INSTALLDIR})
  set (ng-log_DATADIR_DESTINATION \"\${CMAKE_INSTALL_PREFIX}/\${ng-log_DATADIR_DESTINATION}\")
endif (NOT IS_ABSOLUTE ${_ng-log_CMake_INSTALLDIR})

configure_file (\"${ng-log_SOURCE_DIR}/ng-log-modules.cmake.in\"
  \"${ng-log_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/CMakeTmp/ng-log-modules.cmake\" @ONLY)
file (INSTALL
  \"${ng-log_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/CMakeTmp/ng-log-modules.cmake\"
  DESTINATION
  \"\${ng-log_DATADIR_DESTINATION}\")
"
  COMPONENT Development
)

install (FILES
  ${ng-log_BINARY_DIR}/ng-log-config.cmake
  ${ng-log_BINARY_DIR}/ng-log-config-version.cmake
  DESTINATION ${_ng-log_CMake_INSTALLDIR})

# Find modules in share/ng-log/cmake
install (DIRECTORY ${_ng-log_BINARY_CMake_DATADIR}
  DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/ng-log
  COMPONENT Development
  FILES_MATCHING PATTERN "*.cmake"
)

install (EXPORT ng-log-targets NAMESPACE ng-log:: DESTINATION
  ${_ng-log_CMake_INSTALLDIR})
