# SPDX-License-Identifier: LGPL-2.1-or-later

udevadm_sources = files(
        'udevadm.c',
        'udevadm-control.c',
        'udevadm-hwdb.c',
        'udevadm-info.c',
        'udevadm-lock.c',
        'udevadm-monitor.c',
        'udevadm-settle.c',
        'udevadm-test.c',
        'udevadm-test-builtin.c',
        'udevadm-trigger.c',
        'udevadm-util.c',
        'udevadm-verify.c',
        'udevadm-wait.c',
        'udevd.c',
)

libudevd_core_sources = files(
        'udev-ctrl.c',
        'udev-event.c',
        'udev-node.c',
        'udev-rules.c',
        'udev-watch.c',
        'udev-builtin.c',
        'udev-builtin-btrfs.c',
        'udev-builtin-hwdb.c',
        'udev-builtin-input_id.c',
        'udev-builtin-keyboard.c',
        'udev-builtin-net_id.c',
        'udev-builtin-net_setup_link.c',
        'udev-builtin-path_id.c',
        'udev-builtin-usb_id.c',
        'net/link-config.c',
)

if conf.get('HAVE_KMOD') == 1
        libudevd_core_sources += files('udev-builtin-kmod.c')
endif

if conf.get('HAVE_BLKID') == 1
        libudevd_core_sources += files('udev-builtin-blkid.c')
endif

if conf.get('HAVE_ACL') == 1
        libudevd_core_sources += files('udev-builtin-uaccess.c')
endif

############################################################

generate_keyboard_keys_list = find_program('generate-keyboard-keys-list.sh')
keyboard_keys_list_txt = custom_target(
        'keyboard-keys-list.txt',
        output : 'keyboard-keys-list.txt',
        command : [generate_keyboard_keys_list, cpp],
        capture : true)

generate_keyboard_keys_gperf = find_program('generate-keyboard-keys-gperf.sh')
fname = 'keyboard-keys-from-name.gperf'
gperf_file = custom_target(
        fname,
        input : keyboard_keys_list_txt,
        output : fname,
        command : [generate_keyboard_keys_gperf, '@INPUT@'],
        capture : true)

fname = 'keyboard-keys-from-name.h'
keyboard_keys_from_name_h = custom_target(
        fname,
        input : gperf_file,
        output : fname,
        command : [gperf,
                   '-L', 'ANSI-C', '-t',
                   '-N', 'keyboard_lookup_key',
                   '-H', 'hash_key_name',
                   '-p', '-C',
                   '@INPUT@'],
        capture : true)

############################################################

link_config_gperf_c = custom_target(
        'link-config-gperf.c',
        input : 'net/link-config-gperf.gperf',
        output : 'link-config-gperf.c',
        command : [gperf, '@INPUT@', '--output-file', '@OUTPUT@'])

############################################################

if get_option('link-udev-shared')
        udev_link_with = [libshared]
        udev_rpath = rootpkglibdir
else
        udev_link_with = [libshared_static,
                          libsystemd_static]
        udev_rpath = ''
endif

############################################################

udev_includes = [includes, include_directories('net', '.')]
libudevd_core = static_library(
        'udev-core',
        libudevd_core_sources,
        link_config_gperf_c,
        keyboard_keys_from_name_h,
        include_directories : udev_includes,
        link_with : udev_link_with,
        dependencies : [libblkid,
                        libkmod,
                        userspace],
        build_by_default : false)

udev_progs = [['ata_id/ata_id.c'],
              ['cdrom_id/cdrom_id.c'],
              ['fido_id/fido_id.c',
               'fido_id/fido_id_desc.c'],
              ['iocost/iocost.c'],
              ['scsi_id/scsi_id.c',
               'scsi_id/scsi_serial.c'],
              ['v4l_id/v4l_id.c'],
              ['mtd_probe/mtd_probe.c',
               'mtd_probe/probe_smartmedia.c']]

dmi_arches = ['x86', 'x86_64', 'aarch64', 'arm', 'ia64', 'loongarch64', 'mips']
if dmi_arches.contains(host_machine.cpu_family())
        udev_progs += [['dmi_memory_id/dmi_memory_id.c']]
endif

udev_prog_paths = {}
foreach prog : udev_progs
        name = prog[0].split('/')[0]

        exe = executable(
                name,
                prog,
                include_directories : includes,
                dependencies : [userspace,
                                versiondep],
                link_with : udev_link_with,
                install_rpath : udev_rpath,
                install : true,
                install_dir : udevlibexecdir)

        udev_prog_paths += {name : exe}
        public_programs += exe
endforeach

if install_sysconfdir_samples
        install_data('udev.conf',
                     install_dir : sysconfdir / 'udev')
        install_data('iocost/iocost.conf',
                     install_dir : sysconfdir / 'udev')
endif

custom_target(
        'udev.pc',
        input : 'udev.pc.in',
        output : 'udev.pc',
        command : [jinja2_cmdline, '@INPUT@', '@OUTPUT@'],
        install : pkgconfigdatadir != 'no',
        install_dir : pkgconfigdatadir)

if install_sysconfdir
        meson.add_install_script('sh', '-c',
                                 mkdir_p.format(sysconfdir / 'udev/rules.d'))
endif

simple_fuzzers += files(
        'fuzz-udev-rule-parse-value.c',
)

fuzzer_udev_base = {
        'link_with' : [libudevd_core, libshared],
        'dependencies' : [threads, libacl],
}

fuzzers += [
        {
                'sources' : files('net/fuzz-link-parser.c'),
                'includes' : udev_includes,
                'base' : fuzzer_udev_base,
        },
        {
                'sources' : files('fuzz-udev-rules.c'),
                'base' : fuzzer_udev_base,
        },
        {
                'sources' : files(
                        'fido_id/fuzz-fido-id-desc.c',
                        'fido_id/fido_id_desc.c',
                ),
        },
]

test_libudev_base = {
        'link_with' : [libudevd_core, libshared],
        'dependencies' : [threads, libacl],
}

tests += [
        {
                'sources' : files(
                        'fido_id/test-fido-id-desc.c',
                        'fido_id/fido_id_desc.c',
                ),
        },
        {
                'sources' : files('test-udev-builtin.c'),
                'base' : test_libudev_base,
        },
        {
                'sources' : files('test-udev-event.c'),
                'base' : test_libudev_base,
        },
        {
                'sources' : files('test-udev-node.c'),
                'base' : test_libudev_base,
        },
]
