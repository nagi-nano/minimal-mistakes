---
title: "qemu를 사용하여 가상머신에 Windows 10 설치"
date: 2021/07-06
categories:
  - linux
tags:
  - qemu
---

qemu-kvm을 이용해 윈도우 10을 설치 및 사용하는 방법에 대해 정리해 보겠습니다.

## 디스크 이미지 생성 및 윈도우 10 설치

먼저, 디스크 이미지 파일을 생성하여야 합니다. 

``$ qemu-img create -f qcow2 win10.qcow2 100G``

윈도우 10을 설치하기에 앞서 virtio라는 저장장치 및 네트워크 가속 드라이버를 준비하여야 합니다. wget 명령어를 이용해 방금 디스크 이미지를 만든 디렉토리에 받아줍니다.

``$ wget https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso``

다운로드가 완료되면 이제 qemu를 이용해 가상머신을 실행하고 윈도우 10을 설치하시면 됩니다.

``$ qemu-system-x86_64 -cpu host,hv_relaxed,hv_spinlocks=0x1fff,hv_vapic,hv_time -smp cores=4 -m 4096 -nic user,model=virtio-net-pci,id=nic0 -enable-kvm -drive file=win10.qcow2,index=0,media=disk,if=virtio,aio=native,cache.direct=on -drive file='win10-setup.iso',index=2,media=cdrom -drive file=virtio-win.iso,index=3,media=cdrom ``

옵션값을 넣어야 하는데 제가 넣은 설정을 간략하게 정리해 보겠습니다.

  * -cpu host,hv_relaxed,hv_spinlocks=0x1fff,hv_vapic,hv_time : 호스트 cpu의 설정을 그대로 가져오고(kvm 가속을 사용하기 위해) hyper-v 관련 최적화를 적용합니다.
  * -smp cores=4 : cpu 코어 및 스레드를 몇 개 사용할 지에 관한 옵션입니다. 쿼드코어 컴퓨터에서 실행중이므로 4를 넣어 모든 코어를 사용하게 하였습니다.
  * -m 4096 : 가상머신에서 사용할 메모리 용량을 지정합니다. 여기서는 4기가를 사용하게 하였습니다.
  * -nic user,model=virtio-net-pci,id=nic0 : 네트워크 연결에 virtio를 사용합니다.
  * -enable-kvm : kvm 가속을 사용합니다. 
  * -drive file=win10-test.qcow2,index=0,media=disk,if=virtio,aio=native,cache.direct=on : 윈도우 11이 설치될 디스크 파일을 지정합니다.
  * -drive file='win10-setup.iso',index=2,media=cdrom : 윈도우 10 설치 파일이 들어있는 iso파일을 지정합니다.
  * -drive file=virtio-win.iso,index=3,media=cdrom : 아까 받은 virtio 드라이버 iso 파일을 지정합니다.
  
옵션 값을 알맞게 변경하고 qemu를 실행하면 다음과 같이 설치화면이 뜹니다.

![윈도우 10 설치화면](/assets/images/2021/07-06-1.png)

아래 화면에서 사용자 지정으로 설치합니다.

![설치 유형](/assets/images/2021/07-06-2.png)

다음 화면으로 넘어가면 윈도우 설치 파일에서 virtio 드라이버를 찾지 못해서 디스크를 못 찾는다고 에러가 뜨는데 드라이버 로드 옵션으로 들어갑니다.

![드라이버 못 찾음](/assets/images/2021/07-06-3.png)

이 컴퓨터의 하드웨어와 호환되지 않는 드라이버 숨기기의 체크를 풀고 virtio cdrom - amd64 - w10 - 확인을 눌러 드라이버를 찾고 설치합니다.

![드라이버 못 찾음](/assets/images/2021/07-06-4.png)

![드라이버 찾기](/assets/images/2021/07-06-5.png)

이후로는 일반적인 윈도우 설치 과정을 진행합니다.

설치가 완료되면 장치 관리자에서 eternet 컨트롤러를 virtio cdrom에 들어있는 드라이버를 사용해 설치합니다.

설치 완료 후 단독으로 윈도우를 실행할 때는 cdrom 옵션을 빼고 실행하면 됩니다.

```qemu-system-x86_64 -cpu host,hv_relaxed,hv_spinlocks=0x1fff,hv_vapic,hv_time -smp cores=4 -m 4096 -nic user,model=virtio-net-pci,id=nic0 -enable-kvm -drive file=win10.qcow2,media=disk,if=virtio,aio=native,cache.direct=on```

전체 화면 토글은 `Ctrl + Alt + F` 로 하면 되고, 가상머신에 키보드 및 마우스 제어가 잡혀있는 걸 풀고싶으면 `Ctrl + Alt + G` 키 조합을 사용하면 됩니다.

## 설치 및 사용후기

전반적인 사용감은 그 전 virtual box를 사용하였을 때보다 꽤나 괜찮단 느낌입니다. kvm 가속이 되어 속도도 꽤나 빠르구요.. 설정 자체는 레드햇에서 개발하여 배포하는 [Virt-Manager](https://virt-manager.org/)가 gui를 제공하므로 편합니다만 저는 Virt-Manager로 설정할 때 호스트 cpu 정보를 제대로 땡겨오지 못하고 벤치마크 측정시 그냥 qemu로 사용하는 것에 비해 속도가 10% 정도 느리게 나오길래 그냥 qemu로 사용하고 있습니다.

만약 호스트 컴퓨터의 디렉토리를 게스트 컴퓨터에 공유하여 사용하고 싶다면 -nic 옵션 아래에 ```smb=shared_dir_path```를 추가하여 사용하시면 되겠습니다.
