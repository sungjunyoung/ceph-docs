# CEPH STORAGE CLUSTER

ceph storage cluster 는  모든 ceph  배포판에서 기초가 됩니다. RADOS 를 기반으로, ceph storage cluster 는 두가지 종류의 데몬으로 구성되어 있습니다: Ceph OSD Daemon (OSD) 는 스토리지 노드에서 데이터를 오브젝트 형태로 저장하고 있습니다; 그리고, Ceph Monitor (MON) 는 클러스터 맵의 master copy를 유지합니다. ceph storage cluster 는 수천개의 스토리지 노드로 이루어 질 수 있으며, 최소한의 ceph 시스템은 한개의 MON 과 레플리케이션에 필요한 두개의 OSD 데몬으로 이루어져 있습니다.

Ceph Filesystem, Object Storage, Block Device 는 Ceph 스토리지 클러스터로부터 데이터를 읽어들이고, 데이터를 써서 저장합니다.
