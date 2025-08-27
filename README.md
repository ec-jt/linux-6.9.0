#Kernel flags not enough to get rebar working - *machine specific fix*
#root windows 80:03.1 needs 240M unless AST VGA is removed or replace a GPU with a NIC on the root complex, others need 228M
#NP freed by removing BMC VGA (AST) (20MB), SATA AHCI and USB: ASM1042A + AMD xHCI (4MB) and Switchtec mgmt functions

#NP/PF Bridge Memory
lspci -vv -s 00:03.1 | egrep -i 'Memory behind bridge|Prefetchable' -B8 -A5
lspci -vv -s 40:01.1 | egrep -i 'Memory behind bridge|Prefetchable' -B8 -A5
lspci -vv -s 80:03.1 | egrep -i 'Memory behind bridge|Prefetchable' -B8 -A5
lspci -vv -s c0:01.1 | egrep -i 'Memory behind bridge|Prefetchable' -B8 -A5
lspci -tv
lspci -vv | grep -iE -B8 'Memory behind bridge|Prefetchable memory behind bridge'

#Kernel source build
sudo apt-get install -y build-essential bc flex bison libssl-dev libelf-dev dwarves pahole libncurses-dev pkg-config
git clone https://github.com/ec-jt/linux-6.9-g292-z20.git
cd linux-6.9-g292-z20
#config, change max GPU count or peer-to-peer transfer support
cp -v /boot/config-$(uname -r) .config
yes "" | make olddefconfig
scripts/config --disable SYSTEM_TRUSTED_KEYS
scripts/config --disable SYSTEM_REVOCATION_KEYS
scripts/config --disable DEBUG_INFO_BTF
scripts/config --disable DEBUG_INFO_BTF_MODULES
scripts/config --enable PCI_QUIRKS
scripts/config --enable PCI_RESIZABLE_BAR
sudo make olddefconfig
sudo make -j"$(nproc)"
sudo make modules_install install
sudo cp arch/x86/boot/bzImage /boot/vmlinuz-6.9.0
sudo cp System.map /boot/System.map-6.9.0
sudo cp .config /boot/config-6.9.0
sudo update-initramfs -c -k 6.9.0

#update /etc/default/grub with kernel flags https://docs.kernel.org/admin-guide/kernel-parameters.html
GRUB_CMDLINE_LINUX_DEFAULT="pcie_aspm=off intel_iommu=off amd_iommu=off video=efifb:off modprobe.blacklist=ast loglevel=7 \
pcie_ports=native pci=use_crs,realloc=on,assign-busses,big_root_window,hpmmiosize=0"
echo 'blacklist snd_hda_intel' | sudo tee /etc/modprobe.d/blacklist-nvidia-hda.conf
echo 'blacklist snd_hda_codec_hdmi' | sudo tee /etc/modprobe.d/blacklist-nvidia-hda.conf
sudo update-grub
sudo reboot -f

#rebuild/install nvidia kernel
git clone https://github.com/ec-jt/open-gpu-kernel-modules
cd open-gpu-kernel-modules
git checkout 570.148.08-p2p
./install.sh
nvidia-smi topo -p2p r
nvidia-smi topo -m

#debug build errors
#make menuconfig
#make cleanoldconfig 2>/dev/null || true 
#make -j1 V=1 2>&1 | tee build.log

# Change notes
# patched drivers/pci/quirks.c for bar 1 firmware and free NP memory space
/* Pre-size ReBAR for NVIDIA GB202 (RTX 5090) so bridge sizing sees it */
static void quirk_presize_rebar_nvidia_gb202(struct pci_dev *dev)
{
        int bar = 1;               /* NVIDIA FB aperture */
        int sizes, idx, max = 15;  /* cap at 32GB: 2^(15+20) */

        /* only VGA func; audio .1 has no ReBAR/BAR1 */
        if (PCI_FUNC(dev->devfn) != 0)
                return;
        if (!pci_is_pcie(dev))
                return;
        if (!pci_find_ext_capability(dev, PCI_EXT_CAP_ID_REBAR))
                return;

        sizes = pci_rebar_get_possible_sizes(dev, bar);   /* bitmap of indices */
        if (!sizes)
                return;

        /* pick the largest supported index, clamped to 32GB */
        for (idx = min(31, max); idx >= 0; idx--)
                if (sizes & BIT(idx))
                        break;
        if (idx < 0)
                return;

        /* program size early; the later allocator will map windows accordingly */
        if (!pci_rebar_set_size(dev, bar, idx))
                pci_info(dev, "Pre-sized ReBAR on BAR%d to index %d\n", bar, idx);
}
DECLARE_PCI_FIXUP_EARLY(PCI_VENDOR_ID_NVIDIA, 0x2b85, quirk_presize_rebar_nvidia_gb202);
DECLARE_PCI_FIXUP_RESUME_EARLY(PCI_VENDOR_ID_NVIDIA, 0x2b85, quirk_presize_rebar_nvidia_gb202);

/* ==== LAB: free 32-bit NP budget ASAP (EARLY + HEADER) ==== */
static bool lab_bdf(struct pci_dev *d, u8 seg, u8 bus, u8 slot, u8 func)
{
        return pci_domain_nr(d->bus) == seg &&
               d->bus->number == bus &&
               PCI_SLOT(d->devfn) == slot &&
               PCI_FUNC(d->devfn) == func;
} 

static bool lab_np_target(struct pci_dev *d)
{
        /* domain 0000: */
        return
                /* BMC VGA (AST) */
                lab_bdf(d,0,0x84,0x00,0x00) ||
                /* USB: ASM1042A + AMD xHCI */
                lab_bdf(d,0,0x81,0x00,0x00) ||
                lab_bdf(d,0,0x07,0x00,0x03) ||
                lab_bdf(d,0,0x4a,0x00,0x03) ||
                /* Switchtec mgmt functions */
                lab_bdf(d,0,0x02,0x00,0x01) ||
                lab_bdf(d,0,0x41,0x00,0x01) ||
                lab_bdf(d,0,0x86,0,0x01)   ||
                lab_bdf(d,0,0xc1,0,0x01)   ||
                /* SATA AHCI */
                lab_bdf(d,0,0x8c,0x00,0x00);
}

/* EARLY: turn off decode and zero BAR registers so pci_read_bases sees 0 */
static void quirk_lab_free_np_early(struct pci_dev *dev)
{
        if (!lab_np_target(dev))
                return;

        u16 cmd;
        pci_read_config_word(dev, PCI_COMMAND, &cmd);
        if (cmd & (PCI_COMMAND_IO | PCI_COMMAND_MEMORY)) {
                pci_info(dev, "LAB: EARLY disable IO/MEM decode for NP relief\n");
                cmd &= ~(PCI_COMMAND_MASTER | PCI_COMMAND_IO | PCI_COMMAND_MEMORY);
                pci_write_config_word(dev, PCI_COMMAND, cmd);
        }

        /* Zero standard BARs so later sizing sees nothing */
        for (int i = 0; i < PCI_STD_NUM_BARS; i++) {
                u32 bar;
                pci_read_config_dword(dev, PCI_BASE_ADDRESS_0 + 4*i, &bar);
                if (!bar)
                        continue;
                pci_write_config_dword(dev, PCI_BASE_ADDRESS_0 + 4*i, 0);
                if ((bar & PCI_BASE_ADDRESS_SPACE) == PCI_BASE_ADDRESS_SPACE_MEMORY &&
                    (bar & PCI_BASE_ADDRESS_MEM_TYPE_MASK) == PCI_BASE_ADDRESS_MEM_TYPE_64) {
                        /* zero upper dword of 64-bit BAR */
                        i++;
                        pci_write_config_dword(dev, PCI_BASE_ADDRESS_0 + 4*i, 0);
                }
        }
        /* Also disable ROM decode if present */
        pci_write_config_dword(dev, PCI_ROM_ADDRESS, 0);
}
DECLARE_PCI_FIXUP_EARLY(PCI_ANY_ID, PCI_ANY_ID, quirk_lab_free_np_early);

/* HEADER: nuke kernel-side resource flags in case anything slipped through */
static void quirk_lab_free_np_header(struct pci_dev *dev)
{
        if (!lab_np_target(dev))
                return;

        for (int i = 0; i < PCI_STD_NUM_BARS; i++) {
                struct resource *r = &dev->resource[i];
                if (r->flags & (IORESOURCE_MEM | IORESOURCE_IO)) {
                        r->start = 0; r->end = 0; r->flags = 0;
                }
        }
}
DECLARE_PCI_FIXUP_HEADER(PCI_ANY_ID, PCI_ANY_ID, quirk_lab_free_np_header);
/* ==== end LAB NP budget freer ==== */

/*
 * Force selected AMD GPP Root Ports to be treated as hot-plug bridges
 * so that the hot-plug sizing headroom (hpmmiosize/hpprefmemsize/hpbussize)
 * is applied even if the port doesn't advertise HotPlug capability.
 */
static bool quirk_match_bdf(struct pci_dev *pdev, u8 bus, u8 dev, u8 func)
{
        return pdev->bus->number == bus &&
               PCI_SLOT(pdev->devfn) == dev &&
               PCI_FUNC(pdev->devfn) == func;
}

static void quirk_force_hotplug_amd_gpp(struct pci_dev *pdev)
{
        /* Only touch AMD PCIe Root Ports */
        if (!pci_is_pcie(pdev))
                return;
        if (pci_pcie_type(pdev) != PCI_EXP_TYPE_ROOT_PORT)
                return;

        /*
         * Ports to force as hot-plug bridges (example from the logs):
         *   00:03.1, 40:01.1, 80:03.1
         * Edit/add lines as needed (domain assumed 0000:).
         */
        if (quirk_match_bdf(pdev, 0x00, 0x03, 0x01) ||   /* 0000:00:03.1 */
            quirk_match_bdf(pdev, 0x40, 0x01, 0x01) ||   /* 0000:40:01.1 */
            quirk_match_bdf(pdev, 0x80, 0x03, 0x01)) {   /* 0000:80:03.1 */
                pdev->is_hotplug_bridge = true;
                pci_info(pdev, "forcing hot-plug bridge (lab quirk) for sizing\n");
        }
}

/*
 * Starship/Matisse GPP Root Port
 * (We gate inside the function on BDF so we don't flip every 1483.)
 */
#define PCI_DEVICE_ID_AMD_GPP_ROOT_PORT 0x1483
DECLARE_PCI_FIXUP_HEADER(PCI_VENDOR_ID_AMD, PCI_DEVICE_ID_AMD_GPP_ROOT_PORT, quirk_force_hotplug_amd_gpp);

# patched drivers/pci/setup-bus.c force bar 0 allocation
/* Lab override: force a fixed non-prefetchable MEM window on 0000:c0:01.1 */
static inline bool bridge_is_c0_01_1(struct pci_dev *dev)
{
        /* domain 0, bus c0, dev 01, fn 1 */
/*      return dev &&
               pci_domain_nr(dev->bus) == 0 &&
               dev->bus->number == 0xc0 &&
               PCI_SLOT(dev->devfn) == 0x01 &&
               PCI_FUNC(dev->devfn) == 0x01;
*/
        /* domain 0, bus c0|40, dev 01, fn 1 */
/*
        return dev &&
               pci_domain_nr(dev->bus) == 0 &&
               (dev->bus->number == 0xc0 || dev->bus->number == 0x40) &&
               PCI_SLOT(dev->devfn) == 0x01 &&
               PCI_FUNC(dev->devfn) == 0x01;
*/
        return dev &&
               pci_domain_nr(dev->bus) == 0 &&
               PCI_FUNC(dev->devfn) == 0x01 &&
               (
                 /* c0:01.1 */  (dev->bus->number == 0xc0 && PCI_SLOT(dev->devfn) == 0x01) ||
                 /* 00:03.1 */  (dev->bus->number == 0x00 && PCI_SLOT(dev->devfn) == 0x03) ||
                 /* 40:01.1 */  (dev->bus->number == 0x40 && PCI_SLOT(dev->devfn) == 0x01) ||
                 /* 80:03.1 */  (dev->bus->number == 0x80 && PCI_SLOT(dev->devfn) == 0x03)
               );
}

/**
 * pbus_size_mem() - Size the memory window of a given bus
 *
 * @bus:                The bus
 * @mask:               Mask the resource flag, then compare it with type
 * @type:               The type of free resource from bridge
 * @type2:              Second match type
 * @type3:              Third match type
 * @min_size:           The minimum memory window that must be allocated
 * @add_size:           Additional optional memory window
 * @realloc_head:       Track the additional memory window on this list
 *
 * Calculate the size of the bus and minimal alignment which guarantees
 * that all child resources fit in this size.
 *
 * Return -ENOSPC if there's no available bus resource of the desired
 * type.  Otherwise, set the bus resource start/end to indicate the
 * required size, add things to realloc_head (if supplied), and return 0.
 */
static int pbus_size_mem(struct pci_bus *bus, unsigned long mask,
                         unsigned long type, unsigned long type2,
                         unsigned long type3, resource_size_t min_size,
                         resource_size_t add_size,
                         struct list_head *realloc_head)
{
        struct pci_dev *dev;
        resource_size_t min_align, align, size, size0, size1;
        resource_size_t aligns[24]; /* Alignments from 1MB to 8TB */
        int order, max_order;
        struct resource *b_res = find_bus_resource_of_type(bus,
                                        mask | IORESOURCE_PREFETCH, type);
        resource_size_t children_add_size = 0;
        resource_size_t children_add_align = 0;
        resource_size_t add_align = 0;

        if (!b_res)
                return -ENOSPC;

        /* Lab override: if 0000:c0:01.1 non-prefetchable MEM window is
         * already programmed by firmware, release it so we can re-size it. */
        if (bus->self &&
            bridge_is_c0_01_1(bus->self) && 
            b_res == &bus->self->resource[PCI_BRIDGE_MEM_WINDOW] &&
            b_res->parent) {
            pci_info(bus->self,
                     "releasing firmware-assigned non-prefetchable MEM window %pR to re-size\n",
                     b_res);
            release_child_resources(b_res);
            if (!release_resource(b_res))
                pci_dbg(bus->self, "released existing window\n");
            /* keep flags; parent is now NULL so we will size below */
        }

        /* If resource is already assigned, nothing more to do */
        if (b_res->parent)
                return 0;

        memset(aligns, 0, sizeof(aligns));
        max_order = 0;
        size = 0;

        list_for_each_entry(dev, &bus->devices, bus_list) {
                struct resource *r;
                int i;

                pci_dev_for_each_resource(dev, r, i) {
                        const char *r_name = pci_resource_name(dev, i);
                        resource_size_t r_size;

                        if (r->parent || (r->flags & IORESOURCE_PCI_FIXED) ||
                            ((r->flags & mask) != type &&
                             (r->flags & mask) != type2 &&
                             (r->flags & mask) != type3))
                                continue;
                        r_size = resource_size(r);
#ifdef CONFIG_PCI_IOV
                        /* Put SRIOV requested res to the optional list */
                        if (realloc_head && i >= PCI_IOV_RESOURCES &&
                                        i <= PCI_IOV_RESOURCE_END) {
                                add_align = max(pci_resource_alignment(dev, r), add_align);
                                r->end = r->start - 1;
                                add_to_list(realloc_head, dev, r, r_size, 0 /* Don't care */);
                                children_add_size += r_size;
                                continue;
                        }
#endif
                        /*
                         * aligns[0] is for 1MB (since bridge memory
                         * windows are always at least 1MB aligned), so
                         * keep "order" from being negative for smaller
                         * resources.
                         */
                        align = pci_resource_alignment(dev, r);
                        order = __ffs(align) - 20;
                        if (order < 0)
                                order = 0;
                        if (order >= ARRAY_SIZE(aligns)) {
                                pci_warn(dev, "%s %pR: disabling; bad alignment %#llx\n",
                                         r_name, r, (unsigned long long) align);
                                r->flags = 0;
                                continue;
                        }
                        size += max(r_size, align);
                        /*
                         * Exclude ranges with size > align from calculation of
                         * the alignment.
                         */
                        if (r_size <= align)
                                aligns[order] += align;
                        if (order > max_order)
                                max_order = order;

                        if (realloc_head) {
                                children_add_size += get_res_add_size(realloc_head, r);
                                children_add_align = get_res_add_align(realloc_head, r);
                                add_align = max(add_align, children_add_align);
                        }
                }
        }

        min_align = calculate_mem_align(aligns, max_order);
        min_align = max(min_align, window_alignment(bus, b_res->flags));
        size0 = calculate_memsize(size, min_size, 0, 0, resource_size(b_res), min_align);
        add_align = max(min_align, add_align);
        size1 = (!realloc_head || (realloc_head && !add_size && !children_add_size)) ? size0 :
                calculate_memsize(size, min_size, add_size, children_add_size,
                                resource_size(b_res), add_align);
        /*
         * Lab override: for 0000:c0:01.1 force the non-prefetchable MEM window
         * to an exact 240 MiB dont rely on hotplug sizing.
         */
        if (bus->self &&
            bridge_is_c0_01_1(bus->self) &&
            b_res == &bus->self->resource[PCI_BRIDGE_MEM_WINDOW]) {
                resource_size_t forced = 228ULL << 20; /* 228 MiB, 1 MiB granularity */
                size0 = forced;
                size1 = forced;           /* no add_size path */
                add_size = 0;
                children_add_size = 0;
                pci_info(bus->self, "forcing non-prefetchable MEM window to %pa (%llu MiB)\n",
                         &forced, (unsigned long long)(forced >> 20));
        }
        if (!size0 && !size1) {
                if (bus->self && (b_res->start || b_res->end))
                        pci_info(bus->self, "disabling bridge window %pR to %pR (unused)\n",
                                 b_res, &bus->busn_res);
                b_res->flags = 0;
                return 0;
        }
        b_res->start = min_align;
        b_res->end = size0 + min_align - 1;
        b_res->flags |= IORESOURCE_STARTALIGN;
        if (bus->self && size1 > size0 && realloc_head) {
                add_to_list(realloc_head, bus->self, b_res, size1-size0, add_align);
                pci_info(bus->self, "bridge window %pR to %pR add_size %llx add_align %llx\n",
                           b_res, &bus->busn_res,
                           (unsigned long long) (size1 - size0),
                           (unsigned long long) add_align);
        }
        return 0;
}

/*
 * io, mmio and mmio_pref contain the total amount of bridge window space
 * available. This includes the minimal space needed to cover all the
 * existing devices on the bus and the possible extra space that can be
 * shared with the bridges.
 */
static void pci_bus_distribute_available_resources(struct pci_bus *bus,
                                            struct list_head *add_list,
                                            struct resource io,
                                            struct resource mmio,
                                            struct resource mmio_pref)
{
        unsigned int normal_bridges = 0, hotplug_bridges = 0;
        struct resource *io_res, *mmio_res, *mmio_pref_res;
        struct pci_dev *dev, *bridge = bus->self;
        resource_size_t io_per_b, mmio_per_b, mmio_pref_per_b, align;

        io_res = &bridge->resource[PCI_BRIDGE_IO_WINDOW];
        mmio_res = &bridge->resource[PCI_BRIDGE_MEM_WINDOW];
        mmio_pref_res = &bridge->resource[PCI_BRIDGE_PREF_MEM_WINDOW];

        /*
         * The alignment of this bridge is yet to be considered, hence it must
         * be done now before extending its bridge window.
         */
        align = pci_resource_alignment(bridge, io_res);
        if (!io_res->parent && align)
                io.start = min(ALIGN(io.start, align), io.end + 1);

        align = pci_resource_alignment(bridge, mmio_res);
        if (!mmio_res->parent && align)
                mmio.start = min(ALIGN(mmio.start, align), mmio.end + 1);

        align = pci_resource_alignment(bridge, mmio_pref_res);
        if (!mmio_pref_res->parent && align)
                mmio_pref.start = min(ALIGN(mmio_pref.start, align),
                        mmio_pref.end + 1);

        /*
         * Now that we have adjusted for alignment, update the bridge window
         * resources to fill as much remaining resource space as possible.
         */
        adjust_bridge_window(bridge, io_res, add_list, resource_size(&io));
        /* Lab override: don't shrink the non-prefetchable MEM window on 0000:c0:01.1 */
        if (!bridge_is_c0_01_1(bridge)) {
                adjust_bridge_window(bridge, mmio_res, add_list,
                                     resource_size(&mmio));
        }
        adjust_bridge_window(bridge, mmio_pref_res, add_list,
                             resource_size(&mmio_pref));

        /*
         * Calculate how many hotplug bridges and normal bridges there
         * are on this bus.  We will distribute the additional available
         * resources between hotplug bridges.
         */
        for_each_pci_bridge(dev, bus) {
                if (dev->is_hotplug_bridge)
                        hotplug_bridges++;
                else
                        normal_bridges++;
        }

        if (!(hotplug_bridges + normal_bridges))
                return;

        /*
         * Calculate the amount of space we can forward from "bus" to any
         * downstream buses, i.e., the space left over after assigning the
         * BARs and windows on "bus".
         */
        list_for_each_entry(dev, &bus->devices, bus_list) {
                if (!dev->is_virtfn)
                        remove_dev_resources(dev, &io, &mmio, &mmio_pref);
        }

        /*
         * If there is at least one hotplug bridge on this bus it gets all
         * the extra resource space that was left after the reductions
         * above.
         *
         * If there are no hotplug bridges the extra resource space is
         * split between non-hotplug bridges. This is to allow possible
         * hotplug bridges below them to get the extra space as well.
         */
        if (hotplug_bridges) {
                io_per_b = div64_ul(resource_size(&io), hotplug_bridges);
                mmio_per_b = div64_ul(resource_size(&mmio), hotplug_bridges);
                mmio_pref_per_b = div64_ul(resource_size(&mmio_pref),
                                           hotplug_bridges);
        } else {
                io_per_b = div64_ul(resource_size(&io), normal_bridges);
                mmio_per_b = div64_ul(resource_size(&mmio), normal_bridges);
                mmio_pref_per_b = div64_ul(resource_size(&mmio_pref),
                                           normal_bridges);
        }

        for_each_pci_bridge(dev, bus) {
                struct resource *res;
                struct pci_bus *b;

                b = dev->subordinate;
                if (!b)
                        continue;
                if (hotplug_bridges && !dev->is_hotplug_bridge)
                        continue;

                res = &dev->resource[PCI_BRIDGE_IO_WINDOW];

                /*
                 * Make sure the split resource space is properly aligned
                 * for bridge windows (align it down to avoid going above
                 * what is available).
                 */
                align = pci_resource_alignment(dev, res);
                io.end = align ? io.start + ALIGN_DOWN(io_per_b, align) - 1
                               : io.start + io_per_b - 1;

                /*
                 * The x_per_b holds the extra resource space that can be
                 * added for each bridge but there is the minimal already
                 * reserved as well so adjust x.start down accordingly to
                 * cover the whole space.
                 */
                io.start -= resource_size(res);

                res = &dev->resource[PCI_BRIDGE_MEM_WINDOW];
                align = pci_resource_alignment(dev, res);
                mmio.end = align ? mmio.start + ALIGN_DOWN(mmio_per_b, align) - 1
                                 : mmio.start + mmio_per_b - 1;
                mmio.start -= resource_size(res);

                res = &dev->resource[PCI_BRIDGE_PREF_MEM_WINDOW];
                align = pci_resource_alignment(dev, res);
                mmio_pref.end = align ? mmio_pref.start +
                                        ALIGN_DOWN(mmio_pref_per_b, align) - 1
                                      : mmio_pref.start + mmio_pref_per_b - 1;
                mmio_pref.start -= resource_size(res);

                pci_bus_distribute_available_resources(b, add_list, io, mmio,
                                                       mmio_pref);

                io.start += io.end + 1;
                mmio.start += mmio.end + 1;
                mmio_pref.start += mmio_pref.end + 1;
        }
}
