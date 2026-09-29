qemu-system-x86_64 \
  -enable-kvm \
  -m 4096 \
  -smp 2 \
  -drive file=sonic-vs.img,format=qcow2,if=virtio \
  -boot c \
  -netdev user,id=mgmt,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=mgmt \
  -nographic