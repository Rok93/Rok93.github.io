---
layout: post
title: '가상메모리와 페이징 — 8GB 노트북에서 프로그램마다 수백 GB 주소를 쓰는 법'
date: 2026-10-21 09:00:00 +0900
image: /assets/img/virtual-memory-paging/hero.jpg
generated: true
tags: [os, study]
sanitized: true
---

RAM이 8GB인 노트북에서 크롬, IDE, 메신저를 동시에 띄워도 각 프로그램은 "메모리가 부족하다"는 걱정 없이 자기 주소를 마음껏 쓴다. 심지어 64비트 프로그램은 수백 GB짜리 주소 공간을 가진 것처럼 행동한다. 이게 가능한 이유는 프로그램이 보는 주소와 실제 RAM의 주소가 **다르기** 때문이다. 운영체제(OS)와 CPU가 그 사이에서 주소를 **바꿔 치기**한다. 이 글은 그 장치인 **가상메모리(virtual memory)**가 무엇인지, 주소를 어떤 단위로 어떻게 바꾸는지(**페이징, paging**), 바꿀 주소가 RAM에 없을 때 무슨 일이 벌어지는지(**페이지 폴트**)를 차례로 본다. 마지막엔 주소 하나를 손으로 직접 변환하고, 페이지 교체 알고리즘 두 개를 같은 참조열로 돌려 결과를 비교한다.

<p style="font-size:13px;color:#5f5a68;background:#faf8fc;border-left:3px solid #b9a9cc;padding:8px 12px;border-radius:6px;margin:16px 0;">📝 이 글에 나오는 수치·주소·명령 출력은 실제 겪은 장애가 아니라, <b>개념을 쉽게 보여주기 위해 지어낸 예시</b>입니다. (단, 4KB 페이지 크기와 x86-64의 4단계 페이지 테이블 구조는 실제 스펙 그대로입니다.)</p>

<figure style="margin:22px 0;text-align:center;">
<img src="/assets/img/virtual-memory-paging/hero.jpg" alt="보라색 톤의 도서관. 작은 색인 카드 서랍장에서 거대한 서가의 특정 칸으로 화살표가 이어지고, 옆에는 DISK라고 적힌 상자 더미가 놓여 있다" style="max-width:100%;border-radius:12px;">
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">색인 카드(페이지 테이블)만 보면 책(데이터)이 서가(RAM) 어느 칸에 있는지 안다. 서가에 없는 책은 옆의 상자 더미(디스크)에서 꺼내 온다.</figcaption>
</figure>

## 가상메모리 — 프로그램에게 "가짜" 주소 공간을 주는 것

**무엇인가.** 프로그램이 쓰는 주소를 **가상 주소(virtual address)**, RAM 칩에 실제로 꽂히는 주소를 **물리 주소(physical address)**라 부른다. 프로그램은 가상 주소만 본다. CPU 안의 **MMU(Memory Management Unit — 주소 변환 전담 하드웨어)**가 메모리에 접근할 때마다 가상 주소를 물리 주소로 바꾼다. 어떤 가상 주소가 어떤 물리 주소에 대응하는지는 OS가 표로 관리한다.

**왜 중요한가.** 세 가지를 얻는다.

- **격리(isolation)**: 프로세스마다 변환 표가 다르다. A의 가상 주소 `0x1000`과 B의 `0x1000`은 서로 다른 물리 주소다. A가 B의 메모리를 읽을 **경로 자체가 없다**.
- **RAM보다 큰 프로그램**: 당장 안 쓰는 부분은 디스크에 두고 필요할 때만 RAM에 올린다. 프로그램은 자기 데이터가 어디에 있는지 몰라도 된다.
- **공유**: 같은 라이브러리(예: `libc`)를 쓰는 프로세스 100개가 있어도, 각자의 변환 표가 같은 물리 메모리를 가리키게 하면 RAM에는 한 벌만 올라간다.

변환 표를 주소 하나 단위로 만들면 표가 메모리보다 커진다. 그래서 주소를 **덩어리 단위**로 묶어 변환한다. 그 덩어리가 페이지다.

## 페이징 — 메모리를 같은 크기의 칸으로 자른다

**무엇인가.** 가상 주소 공간을 **페이지(page)**라는 고정 크기 조각으로 자른다. 물리 메모리도 같은 크기로 자르고, 이쪽은 **프레임(frame)**이라 부른다. x86-64 리눅스의 기본 페이지 크기는 **4KB(4096바이트)**다. 변환 표는 "가상 페이지 몇 번 → 물리 프레임 몇 번"만 적는다. 이 표가 **페이지 테이블(page table)**이다.

주소 하나는 두 부분으로 나뉜다.

- **VPN(Virtual Page Number — 가상 페이지 번호)**: 상위 비트. 페이지 테이블의 몇 번째 칸을 볼지 정한다.
- **오프셋(offset)**: 하위 비트. 페이지 안에서 몇 번째 바이트인지. 4KB 페이지면 2<sup>12</sup> = 4096이므로 **12비트**다.

변환은 VPN만 바꾸고 **오프셋은 그대로** 붙인다. 페이지 안에서의 위치는 가상이든 물리든 똑같기 때문이다.

<!-- diagram: address-translation -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 190" role="img" aria-label="가상 주소는 가상 페이지 번호와 오프셋으로 나뉜다. 가상 페이지 번호로 페이지 테이블을 찾아 물리 프레임 번호를 얻고, 오프셋은 그대로 붙여 물리 주소를 만든다.">
<text x="80" y="16" font-size="9" fill="#1a1720" font-weight="700" text-anchor="middle">가상 주소</text>
<rect x="15" y="24" width="86" height="26" rx="4" fill="#5F0080"/><text x="58" y="36" font-size="8" fill="#fff" text-anchor="middle">VPN</text><text x="58" y="46" font-size="7" fill="#fff" text-anchor="middle">0x00403</text>
<rect x="101" y="24" width="44" height="26" rx="4" fill="#b9a9cc"/><text x="123" y="36" font-size="8" fill="#1a1720" text-anchor="middle">offset</text><text x="123" y="46" font-size="7" fill="#1a1720" text-anchor="middle">0xA1C</text>
<text x="246" y="16" font-size="9" fill="#1a1720" font-weight="700" text-anchor="middle">페이지 테이블</text>
<rect x="186" y="24" width="120" height="78" rx="6" fill="#fff" stroke="#b9a9cc"/>
<text x="196" y="38" font-size="7" fill="#5f5a68">VPN</text><text x="258" y="38" font-size="7" fill="#5f5a68">PFN</text><text x="290" y="38" font-size="7" fill="#5f5a68">V</text>
<text x="196" y="52" font-size="7" fill="#1a1720">0x00402</text><text x="258" y="52" font-size="7" fill="#1a1720">0x0A11</text><text x="290" y="52" font-size="7" fill="#1f7a4d">1</text>
<rect x="190" y="57" width="112" height="13" rx="3" fill="#e9e1f1"/>
<text x="196" y="66" font-size="7" fill="#5F0080" font-weight="700">0x00403</text><text x="258" y="66" font-size="7" fill="#5F0080" font-weight="700">0x2B7C1</text><text x="290" y="66" font-size="7" fill="#1f7a4d">1</text>
<text x="196" y="80" font-size="7" fill="#1a1720">0x00404</text><text x="258" y="80" font-size="7" fill="#5f5a68">—</text><text x="290" y="80" font-size="7" fill="#b23a00">0</text>
<text x="246" y="95" font-size="6.5" fill="#5f5a68" text-anchor="middle">V=0 이면 RAM에 없음 → 페이지 폴트</text>
<path d="M101 37 L182 37" stroke="#5F0080" stroke-width="1.5" fill="none" marker-end="url(#ar1)"/>
<path d="M246 102 L246 120 L58 120 L58 136" stroke="#5F0080" stroke-width="1.5" fill="none" marker-end="url(#ar1)"/>
<path d="M123 50 L123 136" stroke="#7d5a9e" stroke-width="1.5" stroke-dasharray="3 2" fill="none" marker-end="url(#ar2)"/>
<text x="80" y="156" font-size="9" fill="#1a1720" font-weight="700" text-anchor="middle">물리 주소</text>
<rect x="15" y="160" width="86" height="26" rx="4" fill="#1f7a4d"/><text x="58" y="172" font-size="8" fill="#fff" text-anchor="middle">PFN</text><text x="58" y="182" font-size="7" fill="#fff" text-anchor="middle">0x2B7C1</text>
<rect x="101" y="160" width="44" height="26" rx="4" fill="#b9a9cc"/><text x="123" y="172" font-size="8" fill="#1a1720" text-anchor="middle">offset</text><text x="123" y="182" font-size="7" fill="#1a1720" text-anchor="middle">0xA1C</text>
<text x="246" y="150" font-size="7" fill="#5f5a68" text-anchor="middle">오프셋은 변환하지 않고</text><text x="246" y="161" font-size="7" fill="#5f5a68" text-anchor="middle">그대로 붙인다</text>
<defs><marker id="ar1" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0 0 L6 3 L0 6 Z" fill="#5F0080"/></marker><marker id="ar2" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0 0 L6 3 L0 6 Z" fill="#7d5a9e"/></marker></defs>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">가상 주소의 VPN으로 페이지 테이블을 찾아 PFN(Physical Frame Number — 물리 프레임 번호)을 얻는다. 오프셋은 손대지 않는다. 유효 비트(V)가 0이면 그 페이지는 지금 RAM에 없다.</figcaption>
</figure>

페이지 테이블의 각 칸(**PTE, Page Table Entry**)에는 프레임 번호 말고도 비트 몇 개가 더 들어간다.

- **유효 비트(valid/present)**: 이 페이지가 지금 RAM에 있는가
- **보호 비트**: 읽기·쓰기·실행 허용 여부. 코드 페이지에 쓰기를 시도하면 여기서 걸린다
- **참조 비트·수정 비트(dirty)**: 최근에 읽었는가, 내용이 바뀌었는가. 교체 알고리즘이 쓴다

**왜 고정 크기인가.** 크기가 같으면 "빈 프레임 아무거나" 주면 된다. 크기가 제각각이면 빈 공간이 조각나 쓸 수 없는 틈(**외부 단편화**)이 생긴다. 대신 페이지 안에서 다 못 쓴 끝부분(**내부 단편화**)은 생긴다. 4KB 페이지에 100바이트만 쓰면 나머지는 그냥 논다.

## 변환이 느리면 다 느리다 — TLB와 다단계 페이지 테이블

페이징에는 두 가지 비용 문제가 있다. 하나는 **속도**, 하나는 **테이블 크기**다.

### TLB — 변환 결과를 캐시한다

메모리 접근 한 번에 페이지 테이블도 한 번 읽어야 하면, 모든 접근이 **두 번**이 된다. 그래서 MMU 안에 최근 변환 결과를 담아 두는 작은 캐시를 둔다. **TLB(Translation Lookaside Buffer — 주소 변환 캐시)**다. 보통 수십~천여 개 항목을 담고, 조회는 1사이클 안팎이다.

- **TLB 히트(hit)**: 변환 결과가 TLB에 있다. 페이지 테이블을 안 본다.
- **TLB 미스(miss)**: 없다. 페이지 테이블을 읽어(**페이지 워크, page walk**) 변환하고 결과를 TLB에 넣는다.

프로그램이 좁은 범위의 메모리를 반복해서 쓰면(**지역성, locality**) 히트율이 99%를 넘는다. 반대로 거대한 배열을 무작위로 건너뛰며 접근하면 미스가 쏟아지고, 이때 "메모리는 충분한데 느리다"가 된다.

<!-- diagram: tlb-flow -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 170" role="img" aria-label="CPU가 가상 주소를 내면 먼저 TLB를 본다. 히트면 바로 물리 주소를 얻어 RAM에 접근한다. 미스면 페이지 테이블을 걸어 변환하고 TLB에 채운 뒤 RAM에 접근한다.">
<rect x="10" y="60" width="54" height="40" rx="6" fill="#5F0080"/><text x="37" y="77" font-size="8" fill="#fff" text-anchor="middle">CPU</text><text x="37" y="90" font-size="7" fill="#fff" text-anchor="middle">가상 주소</text>
<path d="M64 80 L96 80" stroke="#5F0080" stroke-width="1.5" fill="none" marker-end="url(#ar3)"/>
<rect x="98" y="52" width="70" height="56" rx="6" fill="#fff" stroke="#7d5a9e"/><text x="133" y="72" font-size="8" fill="#1a1720" font-weight="700" text-anchor="middle">TLB</text><text x="133" y="86" font-size="7" fill="#5f5a68" text-anchor="middle">최근 변환</text><text x="133" y="97" font-size="7" fill="#5f5a68" text-anchor="middle">수십~천여 개</text>
<path d="M168 70 L262 70" stroke="#1f7a4d" stroke-width="1.5" fill="none" marker-end="url(#ar4)"/>
<text x="215" y="63" font-size="7.5" fill="#1f7a4d" font-weight="700" text-anchor="middle">히트 — 1사이클 안팎</text>
<path d="M133 108 L133 136 L172 136" stroke="#b23a00" stroke-width="1.5" fill="none" marker-end="url(#ar5)"/>
<text x="100" y="128" font-size="7.5" fill="#b23a00" font-weight="700">미스</text>
<rect x="174" y="118" width="90" height="36" rx="6" fill="#fff" stroke="#b23a00"/><text x="219" y="132" font-size="8" fill="#1a1720" font-weight="700" text-anchor="middle">페이지 워크</text><text x="219" y="146" font-size="7" fill="#5f5a68" text-anchor="middle">테이블 4단계 읽기</text>
<path d="M264 136 L290 136 L290 102" stroke="#b23a00" stroke-width="1.5" fill="none" marker-end="url(#ar5)"/>
<path d="M219 118 L219 112 L168 112" stroke="#7d5a9e" stroke-width="1" stroke-dasharray="3 2" fill="none" marker-end="url(#ar6)"/>
<text x="208" y="108" font-size="6.5" fill="#7d5a9e" text-anchor="middle">결과를 TLB에 채움</text>
<rect x="264" y="52" width="66" height="50" rx="6" fill="#1f7a4d"/><text x="297" y="72" font-size="8" fill="#fff" text-anchor="middle">RAM</text><text x="297" y="86" font-size="7" fill="#fff" text-anchor="middle">물리 주소 접근</text>
<text x="170" y="18" font-size="9" fill="#1a1720" font-weight="700" text-anchor="middle">메모리 접근 1번의 실제 경로</text>
<text x="170" y="32" font-size="7" fill="#5f5a68" text-anchor="middle">히트가 대부분이어야 페이징 비용이 사라진다</text>
<defs><marker id="ar3" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0 0 L6 3 L0 6 Z" fill="#5F0080"/></marker><marker id="ar4" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0 0 L6 3 L0 6 Z" fill="#1f7a4d"/></marker><marker id="ar5" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0 0 L6 3 L0 6 Z" fill="#b23a00"/></marker><marker id="ar6" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0 0 L6 3 L0 6 Z" fill="#7d5a9e"/></marker></defs>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">TLB 히트면 페이지 테이블을 건드리지 않는다. 미스면 테이블을 여러 단계 읽어야 해서, 미스율이 성능을 좌우한다.</figcaption>
</figure>

### 다단계 페이지 테이블 — 안 쓰는 구간의 표는 아예 안 만든다

32비트 주소에 4KB 페이지면 VPN은 20비트, 즉 페이지가 2<sup>20</sup> = 약 100만 개다. 칸 하나가 4바이트면 **프로세스마다 4MB**의 표가 필요하다. 64비트에서 같은 방식으로 48비트 주소를 다 덮으면 2<sup>36</sup>칸 × 8바이트 = **512GB**다. 표를 통째로 만들 수 없다.

해결은 표를 **트리**로 만드는 것이다. VPN을 여러 조각으로 쪼개서, 첫 조각으로 1단계 표를 찾고, 그 칸이 가리키는 2단계 표를 두 번째 조각으로 찾고… 이렇게 내려간다. 프로세스가 실제로 쓰는 구간의 하위 표만 만들고, 안 쓰는 구간은 상위 칸에 "없음"만 적어 두면 된다. 대부분의 프로세스는 주소 공간의 극히 일부만 쓰므로 표가 수십 KB로 줄어든다.

x86-64는 48비트 가상 주소를 **9비트 × 4단계 + 12비트 오프셋**으로 쪼갠다. 단계 이름은 위에서부터 PML4 → PDPT → PD → PT다. 단계마다 2<sup>9</sup> = 512칸, 칸 하나 8바이트라 표 하나가 딱 4KB, 즉 페이지 한 장에 들어간다. 대신 TLB 미스 한 번에 메모리를 **4번** 읽어야 한다. TLB가 중요한 이유가 여기서도 나온다.

## 페이지 폴트 — 없으면 그때 가져온다

**무엇인가.** 유효 비트가 0인 페이지에 접근하면 MMU가 CPU에 예외를 올린다. 이게 **페이지 폴트(page fault)**다. OS의 폴트 핸들러가 끼어들어 처리하고, 끝나면 프로그램은 아무 일 없었다는 듯 그 명령을 다시 실행한다.

폴트는 "오류"가 아니라 **정상 동작의 일부**다. OS는 프로그램을 시작할 때 코드와 데이터를 전부 RAM에 올리지 않는다. 페이지 테이블에 "없음"만 적어 두고, 실제로 접근할 때 처음 가져온다. 이걸 **요구 페이징(demand paging)**이라 한다. 10GB짜리 프로그램이라도 처음에 실제로 도는 코드가 50MB면 50MB만 올라온다.

폴트는 비용에 따라 두 종류로 나뉜다.

- **마이너 폴트(minor fault)**: 디스크를 안 읽는다. 빈 프레임을 하나 배정해 0으로 채우거나, 이미 RAM에 있는 공유 페이지(라이브러리 등)를 내 테이블에 연결만 한다. 마이크로초 단위.
- **메이저 폴트(major fault)**: 디스크에서 읽어 와야 한다. 실행 파일의 코드 페이지를 처음 읽거나, 스왑으로 쫓겨난 페이지를 되찾을 때. 밀리초 단위, 즉 **천 배 이상** 비싸다.

<!-- diagram: page-fault-flow -->
<figure style="margin:22px 0;padding:16px;background:#faf8fc;border:1px solid #e4e0ec;border-radius:12px;max-width:470px;">
<svg style="width:100%;height:auto;display:block;" viewBox="0 0 340 200" role="img" aria-label="페이지 폴트 처리 흐름. 접근한 페이지의 유효 비트가 0이면 트랩이 발생한다. OS는 빈 프레임을 찾고, 없으면 희생 페이지를 골라 내보낸다. 수정된 페이지면 디스크에 쓴다. 그다음 필요한 페이지를 디스크에서 읽거나 0으로 채우고, 페이지 테이블을 갱신한 뒤 명령을 재실행한다.">
<rect x="10" y="10" width="100" height="30" rx="6" fill="#5F0080"/><text x="60" y="24" font-size="7.5" fill="#fff" text-anchor="middle">① 접근 → V=0</text><text x="60" y="34" font-size="7" fill="#fff" text-anchor="middle">트랩 → OS 핸들러</text>
<path d="M60 40 L60 54" stroke="#5F0080" stroke-width="1.5" fill="none" marker-end="url(#ar7)"/>
<rect x="10" y="56" width="100" height="30" rx="6" fill="#fff" stroke="#7d5a9e"/><text x="60" y="70" font-size="7.5" fill="#1a1720" text-anchor="middle">② 빈 프레임 있나?</text><text x="60" y="80" font-size="7" fill="#5f5a68" text-anchor="middle">free list 확인</text>
<path d="M110 71 L138 71" stroke="#b23a00" stroke-width="1.5" fill="none" marker-end="url(#ar8)"/><text x="124" y="66" font-size="6.5" fill="#b23a00" text-anchor="middle">없음</text>
<rect x="140" y="50" width="100" height="42" rx="6" fill="#fff" stroke="#b23a00"/><text x="190" y="63" font-size="7.5" fill="#1a1720" text-anchor="middle">③ 희생 페이지 선택</text><text x="190" y="74" font-size="7" fill="#5f5a68" text-anchor="middle">(LRU·Clock 등)</text><text x="190" y="86" font-size="7" fill="#b23a00" text-anchor="middle">dirty면 디스크에 먼저 씀</text>
<path d="M190 92 L190 112" stroke="#b23a00" stroke-width="1.5" fill="none" marker-end="url(#ar8)"/>
<path d="M60 86 L60 128 L90 128" stroke="#1f7a4d" stroke-width="1.5" fill="none" marker-end="url(#ar9)"/><text x="40" y="110" font-size="6.5" fill="#1f7a4d">있음</text>
<rect x="92" y="114" width="196" height="30" rx="6" fill="#fff" stroke="#7d5a9e"/><text x="190" y="127" font-size="7.5" fill="#1a1720" text-anchor="middle">④ 프레임 채우기</text><text x="190" y="138" font-size="7" fill="#5f5a68" text-anchor="middle">디스크에서 읽기(메이저) 또는 0으로 채우기(마이너)</text>
<path d="M190 144 L190 158" stroke="#5F0080" stroke-width="1.5" fill="none" marker-end="url(#ar7)"/>
<rect x="92" y="160" width="196" height="30" rx="6" fill="#1f7a4d"/><text x="190" y="173" font-size="7.5" fill="#fff" text-anchor="middle">⑤ PTE 갱신(V=1, PFN 기록) → TLB 갱신</text><text x="190" y="184" font-size="7" fill="#fff" text-anchor="middle">⑥ 폴트 난 명령을 그대로 재실행</text>
<rect x="262" y="10" width="68" height="30" rx="6" fill="#faf8fc" stroke="#b9a9cc"/><text x="296" y="22" font-size="6.5" fill="#5f5a68" text-anchor="middle">디스크 I/O가 끼면</text><text x="296" y="33" font-size="6.5" fill="#b23a00" text-anchor="middle">ms 단위 = 천 배 느림</text>
<defs><marker id="ar7" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0 0 L6 3 L0 6 Z" fill="#5F0080"/></marker><marker id="ar8" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0 0 L6 3 L0 6 Z" fill="#b23a00"/></marker><marker id="ar9" markerWidth="6" markerHeight="6" refX="5" refY="3" orient="auto"><path d="M0 0 L6 3 L0 6 Z" fill="#1f7a4d"/></marker></defs>
</svg>
<figcaption style="font-size:12px;color:#5f5a68;margin-top:8px;">빈 프레임이 없으면 누군가를 내보내야 한다. 그 "누군가"를 고르는 규칙이 페이지 교체 알고리즘이다.</figcaption>
</figure>

### 페이지 교체 — 누구를 내보낼까

RAM이 꽉 찼는데 새 페이지가 필요하면 기존 페이지 하나를 골라 내보낸다(**스왑 아웃, swap out**). 내용이 바뀐 페이지(dirty)면 디스크의 스왑 영역에 먼저 써야 하고, 안 바뀐 페이지면 그냥 버려도 된다(나중에 원본에서 다시 읽으면 되니까). 대표 알고리즘은 다음과 같다.

- **FIFO(First-In First-Out)**: 가장 먼저 들어온 페이지를 내보낸다. 단순하지만 "오래됐어도 자주 쓰는" 페이지를 내보내는 실수를 한다.
- **LRU(Least Recently Used — 가장 오래 안 쓴 것)**: 마지막 접근이 가장 오래된 페이지를 내보낸다. 지역성 가정에 잘 맞는다. 다만 "정확한" LRU는 접근마다 순서를 갱신해야 해서 하드웨어로 구현하기 비싸다.
- **Clock(Second-Chance)**: LRU의 근사. 프레임을 시계 바늘처럼 돌며 참조 비트가 1이면 0으로 내리고 한 번 봐 주고, 0이면 내보낸다. 실제 OS가 쓰는 쪽은 이 계열이다. 리눅스는 active/inactive 두 리스트를 두는 변형을 쓴다.
- **OPT(Optimal)**: "앞으로 가장 늦게 쓸" 페이지를 내보낸다. 미래를 알아야 하므로 실제로는 못 쓰고, 다른 알고리즘의 **비교 기준**으로만 쓴다.

내보냈다가 바로 다시 들여오는 일이 반복되면 CPU가 일은 안 하고 디스크만 기다린다. 이게 **스래싱(thrashing)**이다. 프로세스들이 실제로 활발히 쓰는 페이지 집합(**워킹 셋, working set**)의 합이 RAM보다 커지면 발생한다. 해법은 메모리를 늘리거나 프로세스 수를 줄이는 것뿐이다.

## 상세 예시 ① — 주소 하나를 손으로 변환하기

32비트 주소, 4KB 페이지라고 하자. 오프셋은 12비트(16진수 3자리), VPN은 나머지 20비트(16진수 5자리)다.

이 계산이 하는 일: 가상 주소 `0x00403A1C`를 VPN과 오프셋으로 쪼개고, 페이지 테이블에서 VPN을 찾아 물리 주소를 만든다.

```text
가상 주소        0x00403A1C
               = 0x00403 | 0xA1C
                 ─────┬─   ──┬─
                    VPN      offset (12비트)

페이지 테이블[0x00403] = { PFN: 0x2B7C1, V: 1, RW: 1 }

물리 주소        = PFN  | offset
               = 0x2B7C1 | 0xA1C
               = 0x2B7C1A1C
```

같은 페이지 안의 다른 바이트, 예를 들어 `0x00403000`부터 `0x00403FFF`까지는 **전부 같은 PTE**를 쓴다. 4096바이트 안에서는 변환이 한 번만 일어나고, TLB에도 항목 하나만 들어간다. 배열을 순서대로 훑는 코드가 빠른 이유 중 하나다.

이번엔 `0x00404010`에 접근했다고 하자. VPN은 `0x00404`인데 도식의 테이블에서 이 칸은 V=0이다. MMU가 페이지 폴트를 올리고, OS가 프레임을 배정한 뒤 PTE를 채운다. 프로그램은 그 사실을 모른다. 다만 이 한 번의 접근이 평소보다 수천 배 오래 걸렸을 뿐이다.

## 상세 예시 ② — FIFO와 LRU를 같은 참조열로 돌려 보기

페이지 참조열(프로그램이 접근하는 페이지 번호 순서)을 다음과 같이 두자.

```text
1 2 3 4 1 2 5 1 2 3 4 5
```

프레임이 3개일 때와 4개일 때, FIFO와 LRU로 각각 돌리며 폴트 횟수를 센다. 처음 RAM은 비어 있다.

**FIFO, 프레임 3개.** 들어온 순서대로 내보낸다.

```text
참조  1  2  3  4  1  2  5  1  2  3  4  5
프레임 1  1  1  4  4  4  5  5  5  5  5  5
      -  2  2  2  1  1  1  1  1  3  3  3
      -  -  3  3  3  2  2  2  2  2  4  4
폴트  F  F  F  F  F  F  F  H  H  F  F  H   → 9회
```

**FIFO, 프레임 4개.**

```text
참조  1  2  3  4  1  2  5  1  2  3  4  5
폴트  F  F  F  F  H  H  F  F  F  F  F  F   → 10회
```

프레임을 **늘렸는데 폴트가 늘었다**. 이 역설을 **벨라디의 변칙(Belady's anomaly)**이라 부른다. FIFO는 "오래 있었다"만 보고 "자주 쓴다"는 안 보기 때문에 생긴다.

**LRU, 프레임 3개.** 마지막 접근이 가장 오래된 것을 내보낸다.

```text
참조  1  2  3  4  1  2  5  1  2  3  4  5
폴트  F  F  F  F  F  F  F  H  H  F  F  F   → 10회
```

**LRU, 프레임 4개.**

```text
참조  1  2  3  4  1  2  5  1  2  3  4  5
폴트  F  F  F  F  H  H  F  H  H  F  F  F   → 8회
```

| 알고리즘 | 프레임 3개 | 프레임 4개 |
|---|---|---|
| FIFO | 9회 | **10회** (늘어남) |
| LRU | 10회 | **8회** (줄어듦) |

이 참조열은 벨라디의 변칙을 보여 주려고 일부러 고른 것이라 프레임 3개에선 LRU가 FIFO보다 나쁘다. 하지만 LRU는 프레임을 늘리면 폴트가 **절대 늘지 않는다**(이런 성질을 가진 알고리즘을 스택 알고리즘이라 한다). "메모리를 더 꽂았더니 느려졌다"가 일어나지 않는다는 뜻이고, 실제 OS가 FIFO가 아니라 LRU 계열을 쓰는 이유다.

## 상세 예시 ③ — 리눅스에서 폴트를 직접 세어 보기

마지막으로 요구 페이징이 실제로 "접근할 때" 일어난다는 걸 눈으로 본다.

이 코드가 하는 일: 400MB를 할당만 하고 끝내는 경우와, 할당한 뒤 4KB마다 한 바이트씩 건드리는 경우의 페이지 폴트 수를 비교한다.

```python
# touch.py — 인자로 "alloc" 또는 "touch"
import sys, mmap
SIZE = 400 * 1024 * 1024                 # 400MB
buf = mmap.mmap(-1, SIZE)                # 익명 매핑: 아직 RAM은 안 쓴다
if sys.argv[1] == "touch":
    for off in range(0, SIZE, 4096):     # 페이지마다 1바이트씩
        buf[off] = 1
```

```bash
$ /usr/bin/time -v python3 touch.py alloc 2>&1 | grep -E 'Maximum resident|page faults'
  Maximum resident set size (kbytes): 9856
  Major (requiring I/O) page faults: 0
  Minor (reclaiming a frame) page faults: 1123

$ /usr/bin/time -v python3 touch.py touch 2>&1 | grep -E 'Maximum resident|page faults'
  Maximum resident set size (kbytes): 419712
  Major (requiring I/O) page faults: 0
  Minor (reclaiming a frame) page faults: 103548
```

- `alloc`만 하면 **RSS(Resident Set Size — 실제로 RAM에 올라간 양)**가 10MB 남짓이다. 400MB를 "할당"했지만 OS는 페이지 테이블에 약속만 적어 뒀다.
- `touch`를 하면 RSS가 400MB를 넘고, 마이너 폴트가 약 10만 번 늘었다. 400MB ÷ 4KB = 102,400이니 **페이지 하나당 폴트 하나**가 났다는 뜻이다.
- 메이저 폴트는 0이다. 새로 0으로 채우는 페이지라 디스크를 읽을 일이 없다.

(리눅스에서 THP(Transparent Huge Pages — 2MB짜리 큰 페이지를 자동으로 쓰는 기능)가 켜져 있으면 폴트 수가 수백 번으로 확 줄 수 있다. 큰 페이지 하나가 4KB 페이지 512장 몫을 하기 때문이다.)

프로세스가 "몇 GB를 잡고 있다"는 말은 두 가지 뜻이 될 수 있다. `VmSize`(가상 주소 공간 크기)인지 `VmRSS`(실제 RAM 사용량)인지 구분해야 한다. JVM이 `-Xmx8g`로 떠 있어도 처음엔 RSS가 수백 MB인 이유가 바로 요구 페이징이다.

## 한 줄 교훈

프로그램이 보는 주소는 전부 가짜고, 진짜 RAM은 접근하는 순간 페이지 단위로 뒤늦게 붙는다. 그래서 "메모리 사용량"을 말할 때는 가상 크기인지 실제 상주량인지, 느릴 때는 TLB 미스인지 페이지 폴트인지 먼저 가려야 한다.
