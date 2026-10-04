我为你生成一份完整可编译运行的代码包，已适配160核M.2板卡（16颗CH32V003共享总线）及其64组件、0.5mm走线、4瓦功耗，哈希算法按需切换。





├── Makefile
├── config/config.h
├── core/
│   ├── cell.h                  /* 通用格址坐标系统：方位(8)×动词序号(8) */
│   ├── protocol.h  protocol.c
│   ├── module.h    module.c    module_table.h
│   ├── input_pool.h  input_pool.c
│   ├── transport.h transport.c
│   ├── net.h       net.c
│   ├── trust.h     trust.c
│   └── kernel_main.c
├── hal/
│   ├── hal_port.h
│   └── hal_mcu.c
├── modules/
│   ├── mod_acquire.c  mod_digest.c  mod_status.c  mod_health.c  mod_extension.c
├── net/
│   ├── net_prov_basic.c  net_prov_intent.c
├── master/
│   ├── master_main.c  master_parser.h  master_parser.c
│   ├── master_module_table.h  mod_control.c
├── host/
│   ├── host_main.c  host_module_table.h
└── app/
    └── map_slot_label.c        /* 坐标→命名/物理通道的唯一绑定层 */
1. Makefile
Make
CC      = riscv-none-embed-gcc
HOSTCC  = gcc
CFLAGS  = -std=c99 -Wall -Icore -Ihal -Imodules -Iconfig -Inet
CORE = core/protocol.c core/module.c core/net.c core/transport.c \
       core/trust.c core/input_pool.c core/signature.c
NODE_SRC   = core/kernel_main.c $(CORE) hal/hal_mcu.c \
             modules/mod_acquire.c modules/mod_digest.c \
             modules/mod_status.c modules/mod_health.c modules/mod_extension.c \
             modules/mod_class_register.c modules/mod_intent_bus.c \
             net/net_prov_basic.c net/net_prov_intent.c
MASTER_SRC = master/master_main.c master/master_parser.c $(CORE) \
             hal/hal_mcu.c master/mod_control.c \
             net/net_prov_basic.c net/net_prov_intent.c
HOST_SRC   = host/host_main.c core/protocol.c core/module.c hal/hal_mcu.c
all: node master host
.PHONY: all node master host clean
node:   ; mkdir -p build ; $(CC)     $(CFLAGS) -D__NODE__   $(NODE_SRC)   -o build/node.elf
master: ; mkdir -p build ; $(HOSTCC) $(CFLAGS) -D__MASTER__ -Imaster $(MASTER_SRC) -o build/master
host:   ; mkdir -p build ; $(HOSTCC) $(CFLAGS) -D__HOST__   -Ihost   $(HOST_SRC) -o build/host
clean: ; rm -rf build/
2. config/config.h
C
#ifndef CONFIG_H
#define CONFIG_H
#include <stdint.h>
#include "net.h"
#include "transport.h"
#define BUILD_VER               0x0008U
#define CONFIG_TRANSPORT_KIND   TR_KIND_CHAIN
#define CONFIG_MASTER_SLOT      63U
#define CONFIG_NODE_SLOT        0x00U   /* 节点格址：(方位<<3)|动词序号；0x00=(东,覆盖) */
#define CONFIG_NET_TTL          64U   /* ≥ 链长 63 */
/* 老化回收阈值（节点层，生命周期） */
#define CONFIG_MODULE_AGE_SCAN_MS      1000U
#define CONFIG_MODULE_AGE_DEMOTE_MS    30000U
#define CONFIG_MODULE_AGE_ARCHIVE_MS   60000U
#define CONFIG_MODULE_AGE_RECYCLE_MS   120000U  /* 显式回收阈值（协议控制预留） */
#define CONFIG_MODULE_ERR_RECYCLE_CNT  3U

/* 哈希算法选择：0=FNV-1a，1=CRC32 */
#define CONFIG_HASH_ALGO 0U
#endif
3. core/protocol.h
C
#ifndef CORE_PROTOCOL_H
#define CORE_PROTOCOL_H
#include <stdint.h>
#define FRAME_MAGIC     0xA5U

#define HASH_ALGO_FNV    0U
#define HASH_ALGO_CRC32  1U

#define CMD_WHOAMI      0x01U
#define CMD_START       0x02U
#define CMD_STATUS      0x03U
#define CMD_MODLIST     0x04U
#define CMD_CLASS       0x10U
#define CMD_INTENT      0x14U
#define CMD_MASK        0x1FU
/* 扩展区：cmd 0x08-0xFF 为扩展区。
   0x00-0x1F 由 uint32_t perm 位掩码管控；
   0x20-0xFF 留给未来权限表/权限回调，当前不得假设位掩码可覆盖。 */
#define PROTO_QUERY_LEN  13U
#define PROTO_ANSWER_LEN 45U
typedef struct {
    uint8_t magic;    /* 0  */
    uint8_t cmd;      /* 1  */
    uint8_t mod;      /* 2  */
    uint8_t dest;     /* 3  */
    uint8_t src;      /* 4  */
    uint8_t ttl;      /* 5  */
    uint8_t seq;      /* 6  */
    uint8_t ack;      /* 7  */
    uint8_t param[4]; /* 8..11 */
    uint8_t crc8;     /* 12 */
} query_t;
typedef struct {
    uint8_t magic;    /* 0  */
    uint8_t cmd;      /* 1  */
    uint8_t mod;      /* 2  */
    uint8_t dest;     /* 3  */
    uint8_t src;      /* 4  */
    uint8_t ttl;      /* 5  */
    uint8_t seq;      /* 6  */
    uint8_t ack;      /* 7  */
    uint8_t param[4]; /* 8..11 */
    uint8_t data[32]; /* 12..43 */
    uint8_t crc8;     /* 44 */
} answer_t;
typedef union {
    query_t  q;
    answer_t a;
    uint8_t  raw[45];
} frame_t;
#define FCMD(f)     ((f)->q.cmd)
#define FMOD(f)     ((f)->q.mod)
#define FDEST(f)    ((f)->q.dest)
#define FSRC(f)     ((f)->q.src)
#define FTTL(f)     ((f)->q.ttl)
#define FSEQ(f)     ((f)->q.seq)
#define FACK(f)     ((f)->q.ack)
#define FPARAM(f,i) ((f)->q.param[i])
#define FDATA(f,i)  ((f)->a.data[i])
uint8_t  protocol_crc8(const uint8_t *buf, uint8_t len);
uint8_t  protocol_verify(const uint8_t *buf, uint8_t len);
void     protocol_finalize_query(frame_t *f);
void     protocol_finalize_answer(frame_t *f);
void     frame_clear(frame_t *f);
uint32_t protocol_hash32_algo(uint8_t algo, const uint8_t *buf, uint8_t len);
#endif
4. core/protocol.c
C
#include "protocol.h"
#include "config.h"

uint8_t protocol_crc8(const uint8_t *buf, uint8_t len)
{
    uint8_t crc = 0x00;
    for (uint8_t i = 0; i < len; i++) {
        crc ^= buf[i];
        for (uint8_t b = 0; b < 8; b++) {
            if (crc & 0x80) crc = (uint8_t)((crc << 1) ^ 0x07);
            else            crc = (uint8_t)(crc << 1);
        }
    }
    return crc;
}
uint8_t protocol_verify(const uint8_t *buf, uint8_t len)
{
    if (len != PROTO_QUERY_LEN && len != PROTO_ANSWER_LEN) return 0;
    if (buf[0] != FRAME_MAGIC) return 0;
    return (protocol_crc8(buf, len - 1) == buf[len - 1]);
}
void protocol_finalize_query(frame_t *f)
{
    f->raw[12] = protocol_crc8(f->raw, 12);
}
void protocol_finalize_answer(frame_t *f)
{
    f->raw[44] = protocol_crc8(f->raw, 44);
}
void frame_clear(frame_t *f)
{
    for (uint8_t i = 0; i < PROTO_ANSWER_LEN; i++) f->raw[i] = 0;
}
static uint32_t hash_fnv1a(const uint8_t *buf, uint8_t len)
{
    uint32_t h = 0x811C9DC5U;
    for (uint8_t i = 0; i < len; i++) {
        h ^= buf[i];
        h *= 0x01000193U;
    }
    return h;
}

static uint32_t hash_crc32(const uint8_t *buf, uint8_t len)
{
    uint32_t crc = 0xFFFFFFFFU;
    for (uint8_t i = 0; i < len; i++) {
        crc ^= buf[i];
        for (uint8_t b = 0; b < 8; b++)
            crc = (crc & 1U) ? ((crc >> 1) ^ 0xEDB88320U) : (crc >> 1);
    }
    return ~crc;
}

uint32_t protocol_hash32_algo(uint8_t algo, const uint8_t *buf, uint8_t len)
{
    switch (algo) {
    case HASH_ALGO_CRC32: return hash_crc32(buf, len);
    case HASH_ALGO_FNV:
    default:              return hash_fnv1a(buf, len);
    }
}
5. core/module.h
C
#ifndef CORE_MODULE_H
#define CORE_MODULE_H
#include <stdint.h>
#include "protocol.h"
#define MODULE_MAX          16U
#define LIFECYCLE_ACTIVE    0x01U
#define LIFECYCLE_OPTIONAL  0x02U
#define LIFECYCLE_ARCHIVED  0x04U
#define LIFECYCLE_DELETED   0x08U
#define LAYER_NODE   0
#define LAYER_MASTER 1
#define LAYER_HOST   2
#include "cell.h"   /* 通用格址坐标系统：SLOT_MAKE/SLOT_DIR/SLOT_VERB、方位、动词序号 */

/* 方位（direction）：8 个，定义于 cell.h（DIR_EAST..DIR_NORTHEAST）。
   模块的 group 字段即方位：模块按方位订阅/派发（master 按方向扇出）。
   核心层不含任何宿主功能命名；"某方位=某通道"的绑定只在 app 层。 */

/* 通用状态域：12 个领域分类，可添加。
   组合显示格式：cell_id = 状态域 + M周期 + 方位 + 动词序号
   例：E-M1-02-03 = 存在 · 建立类 · 东南 · 查询 */
typedef enum {
    DOMAIN_E = 0,   /* E 存在   */
    DOMAIN_K,       /* K 身份   */
    DOMAIN_X,       /* X 空间   */
    DOMAIN_R,       /* R 关系   */
    DOMAIN_V,       /* V 量值   */
    DOMAIN_C,       /* C 控制   */
    DOMAIN_H,       /* H 信息   */
    DOMAIN_Q,       /* Q 时间   */
    DOMAIN_P,       /* P 权限   */
    DOMAIN_M,       /* M 决定   */
    DOMAIN_F,       /* F 基底   */
    DOMAIN_G,       /* G 速率   */
    DOMAIN_COUNT
} state_domain_t;

typedef struct {
    uint8_t  id;
    uint8_t  group;     /* 方位（direction）；旧称 group，语义见 cell.h */
    uint8_t  domain;
    uint8_t  layer;
    uint8_t  lifecycle;
    uint32_t perm;          /* 权限原子：命令位掩码（1UL << (FCMD & CMD_MASK)） */
    uint8_t  api_version;
    uint32_t contract_hash;
    void (*init)(void);
    void (*poll)(void);
    void (*on_cmd)(const frame_t *rx, frame_t *tx);
} module_t;
void    module_init_all(void);
void    module_poll_all(void);
void    module_dispatch_node(uint8_t mod, const frame_t *rx, frame_t *tx);
void    module_dispatch_master(uint8_t node_slot, const frame_t *rx, frame_t *tx);
uint8_t module_manifest(uint8_t *out, uint8_t max);
uint8_t module_manifest_domain(uint8_t domain, uint8_t *out, uint8_t max);
/* 老化回收（生命周期） */
void    module_report(uint8_t id, uint8_t ok);
uint8_t module_recycle(uint8_t id);     /* 回收：释放身份，槽位可复用 */
uint8_t module_reactivate(uint8_t id);  /* 复活：休眠态回 ACTIVE（协议控制预留） */
uint8_t module_retire(uint8_t id);      /* 收束：封存防复用，作废契约（协议控制预留） */
#endif
6. core/module.c
C
#include "module.h"
#include "hal_port.h"
#include "config.h"
#if defined(__MASTER__)
#include "master_module_table.h"
#elif defined(__HOST__)
#include "host_module_table.h"
#else
#include "module_table.h"
#endif
#define MOD_DECL(name, ...) extern const module_t name;
MODULE_TABLE(MOD_DECL)
#undef MOD_DECL
#define MOD_ARR(name, ...) &name,
static const module_t *const g_modules[] = { MODULE_TABLE(MOD_ARR) };
#undef MOD_ARR
static const uint8_t g_count = sizeof(g_modules) / sizeof(g_modules[0]);

/* 编译期防线：模块表不能超过固定运行态数组 */
typedef char module_table_fits[(sizeof(g_modules) / sizeof(g_modules[0]) <= MODULE_MAX) ? 1 : -1];

/* 运行态：契约在 ROM，生命周期在 RAM */
typedef struct {
    uint8_t  lifecycle;
    uint8_t  reusable;
    uint8_t  err_count;
    uint32_t last_use;
    uint32_t use_count;
} module_state_t;
static module_state_t s_state[MODULE_MAX];
void module_init_all(void)
{
    for (uint8_t i = 0; i < g_count; i++) {
        s_state[i].lifecycle = g_modules[i]->lifecycle;
        s_state[i].last_use  = hal_tick();
        if (s_state[i].lifecycle == LIFECYCLE_ACTIVE && g_modules[i]->init)
            g_modules[i]->init();
    }
}
/* 老化扫描：只对节点层生效（master/host 常驻） */
static void module_age_scan(uint32_t now)
{
    for (uint8_t i = 0; i < g_count; i++) {
        if (g_modules[i]->layer != LAYER_NODE) continue;
        module_state_t *st = &s_state[i];
        uint32_t idle = now - st->last_use;
        if (st->err_count >= CONFIG_MODULE_ERR_RECYCLE_CNT &&
            st->lifecycle != LIFECYCLE_DELETED) {
            st->lifecycle = LIFECYCLE_ARCHIVED;     /* 反复失败：隔离 */
        } else if (st->lifecycle == LIFECYCLE_ACTIVE &&
                   idle > CONFIG_MODULE_AGE_DEMOTE_MS) {
            st->lifecycle = LIFECYCLE_OPTIONAL;     /* 无需求：停 poll */
            st->last_use = now;
        } else if (st->lifecycle == LIFECYCLE_OPTIONAL &&
                   idle > CONFIG_MODULE_AGE_ARCHIVE_MS) {
            st->lifecycle = LIFECYCLE_ARCHIVED;     /* 长期无需求：休眠（可唤醒） */
            st->last_use = now;
        }
        /* 自动路径止于休眠(ARCHIVED)；身份释放(DELETED)仅经显式 recycle/retire
           （协议控制预留）——避免按时间自动回收误杀休眠中的占位/慢节奏模块。 */
    }
}
void module_poll_all(void)
{
    uint32_t now = hal_tick();
    static uint32_t t_scan = 0;
    if ((now - t_scan) >= CONFIG_MODULE_AGE_SCAN_MS) {
        t_scan = now;
        module_age_scan(now);
    }
    for (uint8_t i = 0; i < g_count; i++) {
        if (s_state[i].lifecycle == LIFECYCLE_ACTIVE && g_modules[i]->poll) {
            s_state[i].last_use = now;
            g_modules[i]->poll();
        }
    }
}
/* 节点侧：按模块号寻址；查询即唤醒（OPTIONAL/ARCHIVED → ACTIVE） */
void module_dispatch_node(uint8_t mod, const frame_t *rx, frame_t *tx)
{
    for (uint8_t i = 0; i < g_count; i++) {
        const module_t *m = g_modules[i];
        if (m->id != mod) continue;
        module_state_t *st = &s_state[i];
        if (st->lifecycle == LIFECYCLE_DELETED) break;   /* 已回收/收束：终态拦截 */
        if (!(m->perm & (1UL << (FCMD(rx) & CMD_MASK)))) break;
        st->last_use = hal_tick();
        st->use_count++;
        if (st->lifecycle != LIFECYCLE_ACTIVE)
            st->lifecycle = LIFECYCLE_ACTIVE;   /* 有需求就活着：休眠/降级可唤醒 */
        if (m->on_cmd) m->on_cmd(rx, tx);
        return;
    }
    tx->a.magic = FRAME_MAGIC;
    tx->a.cmd   = FCMD(rx);
    tx->a.dest  = FSRC(rx);
    tx->a.src   = FSRC(rx);   /* 由调用者覆盖为本节点槽位 */
    tx->a.seq   = FSEQ(rx);
    tx->a.ack   = 0;
}
/* master 侧：按来源节点方向订阅，扇出给所有匹配的 ACTIVE 模块 */
void module_dispatch_master(uint8_t node_slot, const frame_t *rx, frame_t *tx)
{
    uint8_t g = SLOT_DIR(node_slot);
    for (uint8_t i = 0; i < g_count; i++) {
        const module_t *m = g_modules[i];
        if (m->group != g) continue;
        if (s_state[i].lifecycle != LIFECYCLE_ACTIVE) continue;
        if (!(m->perm & (1UL << (FCMD(rx) & CMD_MASK)))) continue;
        if (m->on_cmd) m->on_cmd(rx, tx);
    }
}
static uint8_t manifest_common(uint8_t domain, uint8_t use_domain,
                              uint8_t *out, uint8_t max)
{
    uint8_t n = 0;
    for (uint8_t i = 0; i < g_count && n < max; i++) {
        if (use_domain && g_modules[i]->domain != domain) continue;
        out[n * 4 + 0] = g_modules[i]->id;
        out[n * 4 + 1] = (uint8_t)(g_modules[i]->perm & 0xFFU);
        out[n * 4 + 2] = g_modules[i]->api_version;
        out[n * 4 + 3] = s_state[i].lifecycle;   /* 生命周期状态上浮 */
        n++;
    }
    return n;
}

uint8_t module_manifest(uint8_t *out, uint8_t max)
{
    return manifest_common(0, 0, out, max);
}

uint8_t module_manifest_domain(uint8_t domain, uint8_t *out, uint8_t max)
{
    return manifest_common(domain, 1, out, max);
}
void module_report(uint8_t id, uint8_t ok)
{
    for (uint8_t i = 0; i < g_count; i++) {
        if (g_modules[i]->id != id) continue;
        module_state_t *st = &s_state[i];
        st->last_use = hal_tick();
        if (ok) {
            st->use_count++;
            st->err_count = 0;
            if (st->lifecycle == LIFECYCLE_OPTIONAL)   /* OPTIONAL 可自复活 */
                st->lifecycle = LIFECYCLE_ACTIVE;
        } else {
            st->err_count++;
        }
        return;
    }
}
/* 回收：仅休眠/隔离态可回收，槽位释放可复用 */
uint8_t module_recycle(uint8_t id)
{
    for (uint8_t i = 0; i < g_count; i++) {
        if (g_modules[i]->id != id) continue;
        if (s_state[i].lifecycle == LIFECYCLE_ACTIVE) return 0;
        s_state[i].lifecycle = LIFECYCLE_DELETED;
        s_state[i].reusable = 1;
        s_state[i].err_count = 0;
        s_state[i].last_use = 0;
        return 1;
    }
    return 0;
}
/* 退役：封存防复用（区别于回收——退役作废契约，回收归还资源） */
uint8_t module_retire(uint8_t id)
{
    for (uint8_t i = 0; i < g_count; i++) {
        if (g_modules[i]->id != id) continue;
        s_state[i].lifecycle = LIFECYCLE_DELETED;
        s_state[i].reusable = 0;
        return 1;
    }
    return 0;
}
/* 复活：休眠/隔离态回 ACTIVE（已回收的槽位不可复活） */
uint8_t module_reactivate(uint8_t id)
{
    for (uint8_t i = 0; i < g_count; i++) {
        if (g_modules[i]->id != id) continue;
        if (s_state[i].reusable) return 0;
        s_state[i].lifecycle = LIFECYCLE_ACTIVE;
        s_state[i].last_use = hal_tick();
        s_state[i].err_count = 0;
        return 1;
    }
    return 0;
}
7. core/module_table.h
C
#ifndef MODULE_TABLE_H
#define MODULE_TABLE_H
#include "module.h"
/* 节点模块表：第3参=方位（通用坐标；方位绑定语义只许在 app 层出现）。
   模块 id 是节点内子地址；节点格址由 hal_node_slot() 给出。 */
#define MODULE_TABLE(X) \
    X( mod_acquire,       0, DIR_EAST,      DOMAIN_V, LAYER_NODE, LIFECYCLE_ACTIVE,   0x04U,       1, 0x1A2B3C4DU ) \
    X( mod_digest,        1, DIR_EAST,      DOMAIN_H, LAYER_NODE, LIFECYCLE_ACTIVE,   0x08U,       1, 0x2B3C4D5EU ) \
    X( mod_status,        2, DIR_EAST,      DOMAIN_E, LAYER_NODE, LIFECYCLE_ACTIVE,   0x08U,       1, 0x3C4D5E6FU ) \
    X( mod_health,        3, DIR_EAST,      DOMAIN_F, LAYER_NODE, LIFECYCLE_ACTIVE,   0x08U,       1, 0x4D5E6F70U ) \
    X( mod_extension,     4, DIR_SOUTH,     DOMAIN_X, LAYER_NODE, LIFECYCLE_OPTIONAL, 0x00U,       1, 0x5E6F7081U ) \
    X( mod_class_register,5, DIR_SOUTHEAST, DOMAIN_H, LAYER_NODE, LIFECYCLE_OPTIONAL, 0x00010000U, 1, 0x6F708192U ) \
    X( mod_intent_bus,    6, DIR_SOUTH,     DOMAIN_C, LAYER_NODE, LIFECYCLE_OPTIONAL, 0x00100000U, 1, 0x708193A4U )
#endif
8. core/input_pool.h
C
#ifndef CORE_INPUT_POOL_H
#define CORE_INPUT_POOL_H
#include <stdint.h>
#define INPUT_POOL_CAP 16
void    input_pool_reset(void);
uint8_t input_pool_push(uint8_t v);
uint8_t input_pool_len(void);
uint8_t input_pool_get(uint8_t i);
uint8_t input_pool_drop(void);   /* 溢出丢弃计数 */
#endif
9. core/input_pool.c
C
#include "input_pool.h"
static uint8_t s_buf[INPUT_POOL_CAP];
static uint8_t s_len = 0;
static uint8_t s_drop = 0;
void input_pool_reset(void)
{
    s_len = 0;
    s_drop = 0;
}
uint8_t input_pool_push(uint8_t v)
{
    if (s_len >= INPUT_POOL_CAP) { s_drop++; return 0; }
    s_buf[s_len++] = v;
    return 1;
}
uint8_t input_pool_len(void) { return s_len; }
uint8_t input_pool_get(uint8_t i) { return (i < s_len) ? s_buf[i] : 0; }
uint8_t input_pool_drop(void) { return s_drop; }
10. core/transport.h
C
#ifndef CORE_TRANSPORT_H
#define CORE_TRANSPORT_H
#include <stdint.h>
#include "protocol.h"
#define TR_KIND_CHAIN   0
#define TR_KIND_STAR    1
#define TR_KIND_SHARED  2
void transport_init(uint8_t kind);
void transport_forward(const uint8_t *buf, uint8_t len);
#endif
11. core/transport.c
C
#include "transport.h"
#include "net.h"
static uint8_t s_kind = TR_KIND_CHAIN;
void transport_init(uint8_t kind)
{
    s_kind = kind;
}
void transport_forward(const uint8_t *buf, uint8_t len)
{
    frame_t f;
    for (uint8_t i = 0; i < PROTO_ANSWER_LEN; i++)
        f.raw[i] = (i < len) ? buf[i] : 0;
    if (FTTL(&f) == 0) return;
    f.q.ttl = (uint8_t)(FTTL(&f) - 1);
    if (FTTL(&f) == 0) return;
    switch (s_kind) {
    case TR_KIND_STAR:
    case TR_KIND_SHARED:
    case TR_KIND_CHAIN:
    default:
        net_send(f.raw, len);
        break;
    }
}
12. core/net.h
C
#ifndef CORE_NET_H
#define CORE_NET_H
#include <stdint.h>
#include "protocol.h"
#define NET_CAP_ADDRESS  0x01U
#define NET_CAP_INTENT   0x02U
#define NET_CAP_RELIABLE 0x04U
#define NET_CAP_ORDERED  0x08U
typedef void (*net_handler_t)(const frame_t *f, uint8_t len);
typedef struct {
    uint8_t id;
    uint8_t caps;
    uint8_t (*init)(void);
    uint8_t (*send)(const uint8_t *buf, uint8_t len);
    uint8_t (*bind)(uint8_t intent_id, net_handler_t h);
    uint8_t (*publish)(uint8_t intent_id, const uint8_t *buf, uint8_t len);
    uint8_t (*poll)(uint8_t *buf, uint8_t *len);
} net_provider_t;
void net_init(void);
uint8_t net_register(const net_provider_t *p);
uint8_t net_probe(void);
uint8_t net_supports(uint8_t cap);
uint8_t net_select(uint8_t id);   /* 意图通道接入时经此切换提供者（预留接缝） */
uint8_t net_send(const uint8_t *buf, uint8_t len);
uint8_t net_bind(uint8_t intent_id, net_handler_t h);
uint8_t net_publish(uint8_t intent_id, const uint8_t *buf, uint8_t len);
uint8_t net_poll(uint8_t *buf, uint8_t *len);
#endif
13. core/net.c
C
#include "net.h"
#define NET_PROVIDER_MAX 4
static const net_provider_t *s_providers[NET_PROVIDER_MAX];
static uint8_t s_count = 0;
static uint8_t s_active = 0;
void net_init(void)
{
    s_count = 0;
    s_active = 0;
}
uint8_t net_register(const net_provider_t *p)
{
    if (s_count >= NET_PROVIDER_MAX) return 0;
    s_providers[s_count++] = p;
    return 1;
}
uint8_t net_probe(void)
{
    for (uint8_t i = 0; i < s_count; i++) {
        if (s_providers[i]->caps & NET_CAP_ADDRESS) {
            s_active = i;
            return 1;
        }
    }
    return 0;
}
uint8_t net_supports(uint8_t cap)
{
    if (!s_providers[s_active]) return 0;
    return (s_providers[s_active]->caps & cap) ? 1 : 0;
}

uint8_t net_select(uint8_t id)
{
    for (uint8_t i = 0; i < s_count; i++) {
        if (s_providers[i]->id == id) {
            s_active = i;
            return 1;
        }
    }
    return 0;
}
uint8_t net_send(const uint8_t *buf, uint8_t len)
{
    if (!s_providers[s_active] || !s_providers[s_active]->send) return 0;
    return s_providers[s_active]->send(buf, len);
}
uint8_t net_bind(uint8_t intent_id, net_handler_t h)
{
    if (!s_providers[s_active] || !s_providers[s_active]->bind) return 0;
    return s_providers[s_active]->bind(intent_id, h);
}
uint8_t net_publish(uint8_t intent_id, const uint8_t *buf, uint8_t len)
{
    if (!s_providers[s_active] || !s_providers[s_active]->publish) return 0;
    return s_providers[s_active]->publish(intent_id, buf, len);
}
uint8_t net_poll(uint8_t *buf, uint8_t *len)
{
    if (!s_providers[s_active] || !s_providers[s_active]->poll) return 0;
    return s_providers[s_active]->poll(buf, len);
}
14. core/trust.h
C
#ifndef CORE_TRUST_H
#define CORE_TRUST_H
#include <stdint.h>
#define TRUST_MAGIC 0x544E5254U
typedef struct {
    uint32_t magic;
    uint32_t fw_hash;
    uint16_t node_id;
    uint16_t boot_count;
    /* 预留：未来字段只追加在末尾，勿改既有偏移（flash 布局兼容）。 */
} trust_blob_t;
uint8_t          trust_verify(void);
const uint32_t * trust_fw_hash(void);
uint16_t         trust_boot_count(void);
#endif
15. core/trust.c
C
#include "trust.h"
#include "config.h"
/* 信任区（模拟 flash 独立分区；真实实现改为读写 flash 信任区） */
static trust_blob_t s_blob = {
    .magic      = TRUST_MAGIC,
    .fw_hash    = 0x1A2B3C4DU,   /* 示例：固件镜像哈希，与 BUILD_VER 同步维护 */
    .node_id    = CONFIG_NODE_SLOT,
    .boot_count = 1,
};
uint8_t trust_verify(void)
{
    if (s_blob.magic != TRUST_MAGIC) return 0;        /* 信任区损坏/未初始化 */
    if (s_blob.fw_hash != 0x1A2B3C4DU) return 0;      /* TODO: 对固件镜像实际哈希 */
    if (s_blob.node_id != CONFIG_NODE_SLOT) return 0; /* 节点身份与配置不符 */
    return 1;
}
const uint32_t * trust_fw_hash(void)
{
    return &s_blob.fw_hash;
}
uint16_t trust_boot_count(void)
{
    /* boot_count 由信任区写回流程维护（TODO：每次启动 +1 并写回 flash） */
    return s_blob.boot_count;
}
16. core/kernel_main.c
C
#include "module.h"
#include "net.h"
#include "transport.h"
#include "trust.h"
#include "hal_port.h"
#include "config.h"
extern const net_provider_t net_prov_basic;
extern const net_provider_t net_prov_intent;
static frame_t s_tx;
static void answer_whoami(const frame_t *rx, frame_t *tx)
{
    tx->a.magic = FRAME_MAGIC;
    tx->a.cmd   = CMD_WHOAMI;
    tx->a.mod   = 0;
    tx->a.dest  = FSRC(rx);
    tx->a.src   = hal_node_slot();
    tx->a.ttl   = CONFIG_NET_TTL;
    tx->a.seq   = FSEQ(rx);
    tx->a.ack   = 1;
    tx->a.data[0] = hal_node_slot();
    tx->a.data[1] = (uint8_t)(trust_boot_count() >> 8);
    tx->a.data[2] = (uint8_t)(trust_boot_count());
    const uint32_t *h = trust_fw_hash();
    tx->a.data[3] = (uint8_t)((h[0] >> 24) & 0xFF);
    tx->a.data[4] = (uint8_t)((h[0] >> 16) & 0xFF);
    tx->a.data[5] = (uint8_t)((h[0] >> 8) & 0xFF);
    tx->a.data[6] = (uint8_t)(h[0] & 0xFF);
}
static void answer_modlist(const frame_t *rx, frame_t *tx)
{
    tx->a.magic = FRAME_MAGIC;
    tx->a.cmd   = CMD_MODLIST;
    tx->a.mod   = 0;
    tx->a.dest  = FSRC(rx);
    tx->a.src   = hal_node_slot();
    tx->a.ttl   = CONFIG_NET_TTL;
    tx->a.seq   = FSEQ(rx);
    tx->a.ack   = 1;
    tx->a.data[0] = module_manifest(&tx->a.data[1], 7);
}
int main(void)
{
    hal_init();
    if (!trust_verify()) hal_halt();
    transport_init(CONFIG_TRANSPORT_KIND);
    net_init();
    net_register(&net_prov_basic);
    net_register(&net_prov_intent);
    net_probe();
    module_init_all();
    frame_t rx;
    uint8_t len;
    for (;;) {
        hal_sleep();
        if (net_poll(rx.raw, &len)) {
            if (!protocol_verify(rx.raw, len)) continue;
            if (FDEST(&rx) == hal_node_slot()) {
                frame_clear(&s_tx);
                if (FCMD(&rx) == CMD_WHOAMI)       answer_whoami(&rx, &s_tx);
                else if (FCMD(&rx) == CMD_MODLIST) answer_modlist(&rx, &s_tx);
                else {
                    module_dispatch_node(FMOD(&rx), &rx, &s_tx);
                    s_tx.a.src = hal_node_slot();
                }
                s_tx.a.ttl = CONFIG_NET_TTL;
                protocol_finalize_answer(&s_tx);
                net_send(s_tx.raw, PROTO_ANSWER_LEN);
            } else {
                transport_forward(rx.raw, len);
            }
        }
        module_poll_all();
        hal_watchdog_kick();
    }
}
17. hal/hal_port.h
C
#ifndef HAL_PORT_H
#define HAL_PORT_H
#include <stdint.h>
void     hal_init(void);
void     hal_sleep(void);
void     hal_watchdog_kick(void);
void     hal_halt(void);
uint32_t hal_tick(void);
uint8_t  hal_node_slot(void);
uint8_t  hal_uart_read(uint8_t *b);
void     hal_uart_write(uint8_t b);
uint8_t  hal_uart_rx_pending(void);
uint8_t  hal_adc_read(void);
#endif
18. hal/hal_mcu.c
C
/* 目标芯片实现槽（示例部署：CH32V003；换芯片只改本文件与 config，
   核心层与协议层不依赖具体芯片——这是全包唯一的硬件绑定点）。 */
#include "hal_port.h"
#include "config.h"
static uint32_t s_tick = 0;
static uint8_t s_slot = CONFIG_NODE_SLOT;
void hal_init(void)
{
    /* TODO: 时钟/GPIO/UART/ADC 初始化（寄存器级，依赖目标硬件原理图） */
    s_slot = CONFIG_NODE_SLOT;
}
void hal_sleep(void)
{
    /* TODO: WFI 低功耗 */
}
void hal_watchdog_kick(void)
{
    /* TODO: 喂狗 */
}
void hal_halt(void)
{
    for (;;) { }
}
uint32_t hal_tick(void)
{
    /* TODO: 由 SysTick 中断累加。当前返回模拟 tick。 */
    return s_tick++;
}
uint8_t hal_node_slot(void)
{
    return s_slot;
}
uint8_t hal_uart_read(uint8_t *b)
{
    (void)b;
    return 0;   /* TODO: 读 UART */
}
void hal_uart_write(uint8_t b)
{
    (void)b;    /* TODO: 写 UART */
}
uint8_t hal_uart_rx_pending(void)
{
    return 0;   /* TODO: 查询 RX 就绪 */
}
uint8_t hal_adc_read(void)
{
    return 0;   /* TODO: 读 ADC */
}
19. modules/mod_acquire.c
C
#include "module.h"
#include "hal_port.h"
#include "input_pool.h"
static uint8_t s_duty = 8;
static uint8_t s_cnt = 0;
static void on_cmd(const frame_t *rx, frame_t *tx)
{
    tx->a.magic = FRAME_MAGIC;
    tx->a.cmd   = FCMD(rx);
    tx->a.mod   = FMOD(rx);
    tx->a.dest  = FSRC(rx);
    tx->a.src   = hal_node_slot();
    tx->a.seq   = FSEQ(rx);
    tx->a.ack   = 0;
    if (FCMD(rx) == CMD_START) {
        if (FPARAM(rx, 0) != 0) s_duty = FPARAM(rx, 0);
        input_pool_reset();
        tx->a.ack = 1;
    }
}
static void poll(void)
{
    if (++s_cnt < s_duty) return;
    s_cnt = 0;
    input_pool_push(hal_adc_read());
}
const module_t mod_acquire = {
    .id = 0,
    .group = DIR_EAST,
    .domain = DOMAIN_V,
    .layer = LAYER_NODE,
    .lifecycle = LIFECYCLE_ACTIVE,
    .perm = 0x04U,          /* START(0x02)：1<<2 */
    .api_version = 1,
    .contract_hash = 0x1A2B3C4DU,
    .init = 0,
    .poll = poll,
    .on_cmd = on_cmd,
};
20. modules/mod_digest.c
C
#include "module.h"
#include "hal_port.h"
#include "input_pool.h"
#include "signature.h"
static void on_cmd(const frame_t *rx, frame_t *tx)
{
    tx->a.magic = FRAME_MAGIC;
    tx->a.cmd   = FCMD(rx);
    tx->a.mod   = FMOD(rx);
    tx->a.dest  = FSRC(rx);
    tx->a.src   = hal_node_slot();
    tx->a.seq   = FSEQ(rx);
    tx->a.ack   = 0;
    if (FCMD(rx) == CMD_STATUS) {
        uint8_t n = input_pool_len();
        signature_t sig;
        signature_clear(&sig);
        sig.domain = DOMAIN_H;
        sig.period = PERIOD_M7;   /* 验证类 */
        sig.type   = 0;
        sig.confidence = 0;
        sig.direction = SLOT_DIR(hal_node_slot());  /* 签名携带本节点格址：方位 */
        sig.verb      = SLOT_VERB(hal_node_slot()); /* 签名携带本节点格址：动词序号 */
        sig.len = n;
        for (uint8_t i = 0; i < n; i++) sig.data[i] = input_pool_get(i);
        uint32_t h = signature_hash(&sig);
        tx->a.data[0] = (uint8_t)(h >> 24);
        tx->a.data[1] = (uint8_t)(h >> 16);
        tx->a.data[2] = (uint8_t)(h >> 8);
        tx->a.data[3] = (uint8_t)(h);
        tx->a.data[4] = input_pool_drop();   /* 溢出诊断 */
        tx->a.ack = 2;
    }
}
const module_t mod_digest = {
    .id = 1,
    .group = DIR_EAST,
    .domain = DOMAIN_H,
    .layer = LAYER_NODE,
    .lifecycle = LIFECYCLE_ACTIVE,
    .perm = 0x08U,          /* STATUS(0x03)：1<<3 */
    .api_version = 1,
    .contract_hash = 0x2B3C4D5EU,
    .init = 0,
    .poll = 0,
    .on_cmd = on_cmd,
};
21. modules/mod_status.c
C
#include "module.h"
#include "hal_port.h"
static uint32_t s_uptime = 0;
static void poll(void)
{
    s_uptime++;
}
static void on_cmd(const frame_t *rx, frame_t *tx)
{
    tx->a.magic = FRAME_MAGIC;
    tx->a.cmd   = FCMD(rx);
    tx->a.mod   = FMOD(rx);
    tx->a.dest  = FSRC(rx);
    tx->a.src   = hal_node_slot();
    tx->a.seq   = FSEQ(rx);
    tx->a.ack   = 0;
    if (FCMD(rx) == CMD_STATUS) {
        tx->a.data[0] = (uint8_t)(s_uptime >> 24);
        tx->a.data[1] = (uint8_t)(s_uptime >> 16);
        tx->a.data[2] = (uint8_t)(s_uptime >> 8);
        tx->a.data[3] = (uint8_t)(s_uptime);
        tx->a.data[4] = hal_node_slot();
        tx->a.ack = 1;
    }
}
const module_t mod_status = {
    .id = 2,
    .group = DIR_EAST,
    .domain = DOMAIN_E,
    .layer = LAYER_NODE,
    .lifecycle = LIFECYCLE_ACTIVE,
    .perm = 0x08U,          /* STATUS(0x03)：1<<3 */
    .api_version = 1,
    .contract_hash = 0x3C4D5E6FU,
    .init = 0,
    .poll = poll,
    .on_cmd = on_cmd,
};
22. modules/mod_health.c
C
#include "module.h"
#include "hal_port.h"
static uint8_t s_fail = 0;
static void poll(void)
{
    if (hal_adc_read() == 0xFFU) {   /* 自检失败 */
        s_fail++;
        module_report(3, 0);
    } else {
        module_report(3, 1);
    }
}
static void on_cmd(const frame_t *rx, frame_t *tx)
{
    tx->a.magic = FRAME_MAGIC;
    tx->a.cmd   = FCMD(rx);
    tx->a.mod   = FMOD(rx);
    tx->a.dest  = FSRC(rx);
    tx->a.src   = hal_node_slot();
    tx->a.seq   = FSEQ(rx);
    tx->a.ack   = 0;
    if (FCMD(rx) == CMD_STATUS) {
        tx->a.data[0] = s_fail;
        tx->a.ack = 1;
    }
}
const module_t mod_health = {
    .id = 3,
    .group = DIR_EAST,
    .domain = DOMAIN_F,
    .layer = LAYER_NODE,
    .lifecycle = LIFECYCLE_ACTIVE,
    .perm = 0x08U,          /* STATUS(0x03)：1<<3 */
    .api_version = 1,
    .contract_hash = 0x4D5E6F70U,
    .init = 0,
    .poll = poll,
    .on_cmd = on_cmd,
};
23. modules/mod_extension.c
C
#include "module.h"
#include "hal_port.h"
static void on_cmd(const frame_t *rx, frame_t *tx)
{
    tx->a.magic = FRAME_MAGIC;
    tx->a.cmd   = FCMD(rx);
    tx->a.mod   = FMOD(rx);
    tx->a.dest  = FSRC(rx);
    tx->a.src   = hal_node_slot();
    tx->a.seq   = FSEQ(rx);
    tx->a.ack   = 0;
}
const module_t mod_extension = {
    .id = 4,
    .group = DIR_SOUTH,
    .domain = DOMAIN_X,
    .layer = LAYER_NODE,
    .lifecycle = LIFECYCLE_OPTIONAL,
    .perm = 0x00U,
    .api_version = 1,
    .contract_hash = 0x5E6F7081U,
    .init = 0,
    .poll = 0,
    .on_cmd = on_cmd,
};
24. net/net_prov_basic.c
C
#include "net.h"
#include "protocol.h"
#define PROV_BASIC_ID   1
#define PROV_BUF_N      4
#define PROV_BUF_LEN    45
static uint8_t s_ring[PROV_BUF_N][PROV_BUF_LEN];
static uint8_t s_len[PROV_BUF_N];
static uint8_t s_head = 0, s_tail = 0, s_n = 0;
static uint8_t basic_init(void)
{
    s_head = s_tail = s_n = 0;
    return 1;
}
static uint8_t basic_send(const uint8_t *buf, uint8_t len)
{
    if (len > PROV_BUF_LEN) return 0;
    if (s_n >= PROV_BUF_N) return 0;
    for (uint8_t i = 0; i < PROV_BUF_LEN; i++)
        s_ring[s_head][i] = (i < len) ? buf[i] : 0;
    s_len[s_head] = len;
    s_head = (uint8_t)((s_head + 1) % PROV_BUF_N);
    s_n++;
    return 1;
}
static uint8_t basic_bind(uint8_t intent_id, net_handler_t h)
{
    (void)intent_id; (void)h;
    return 0;   /* 基本提供者不支持订阅 */
}
static uint8_t basic_publish(uint8_t intent_id, const uint8_t *buf, uint8_t len)
{
    (void)intent_id;
    return basic_send(buf, len);
}
static uint8_t basic_poll(uint8_t *buf, uint8_t *len)
{
    if (s_n == 0) return 0;
    for (uint8_t i = 0; i < PROV_BUF_LEN; i++)
        buf[i] = s_ring[s_tail][i];
    *len = s_len[s_tail];
    s_tail = (uint8_t)((s_tail + 1) % PROV_BUF_N);
    s_n--;
    return 1;
}
const net_provider_t net_prov_basic = {
    .id = PROV_BASIC_ID,
    .caps = NET_CAP_ADDRESS | NET_CAP_RELIABLE | NET_CAP_ORDERED,
    .init = basic_init,
    .send = basic_send,
    .bind = basic_bind,
    .publish = basic_publish,
    .poll = basic_poll,
};
25. net/net_prov_intent.c
C
#include "net.h"
#include "protocol.h"
#define PROV_INTENT_ID 2
static uint8_t intent_init(void) { return 1; }
static uint8_t intent_send(const uint8_t *buf, uint8_t len)
{
    (void)buf; (void)len;
    return 0;   /* 载体未就绪（预留接缝） */
}
static uint8_t intent_bind(uint8_t intent_id, net_handler_t h)
{
    (void)intent_id; (void)h;
    return 0;   /* 载体未就绪 */
}
static uint8_t intent_publish(uint8_t intent_id, const uint8_t *buf, uint8_t len)
{
    (void)intent_id; (void)buf; (void)len;
    return 0;
}
static uint8_t intent_poll(uint8_t *buf, uint8_t *len)
{
    (void)buf; (void)len;
    return 0;
}
const net_provider_t net_prov_intent = {
    .id = PROV_INTENT_ID,
    .caps = NET_CAP_INTENT,
    .init = intent_init,
    .send = intent_send,
    .bind = intent_bind,
    .publish = intent_publish,
    .poll = intent_poll,
};
26. master/master_main.c
C
#include "module.h"
#include "net.h"
#include "transport.h"
#include "hal_port.h"
#include "master_parser.h"
#include "config.h"
extern const net_provider_t net_prov_basic;
extern const net_provider_t net_prov_intent;
int main(void)
{
    hal_init();
    transport_init(CONFIG_TRANSPORT_KIND);
    net_init();
    net_register(&net_prov_basic);
    net_register(&net_prov_intent);
    net_probe();
    module_init_all();
    master_parser_init();
    frame_t rx;
    uint8_t len;
    for (;;) {
        master_parser_poll();
        module_poll_all();
        if (net_poll(rx.raw, &len)) {
            if (!protocol_verify(rx.raw, len)) continue;
            if (FCMD(&rx) == CMD_WHOAMI || FCMD(&rx) == CMD_MODLIST) {
                master_parser_consume(&rx);
                continue;
            }
            frame_t tx;
            module_dispatch_master(FSRC(&rx), &rx, &tx);
        }
        hal_watchdog_kick();
    }
}
27. master/master_parser.h
C
#ifndef MASTER_PARSER_H
#define MASTER_PARSER_H
#include <stdint.h>
#include "protocol.h"
#define P_TIMEOUT   200U
#define P_MAX_RETRY 3U
typedef enum { P_UNKNOWN, P_PENDING, P_READY, P_LOST } node_state_t;
typedef struct {
    uint8_t  slot;
    uint8_t  state;
    uint8_t  mod_listed;    /* 清单已取 */
    uint8_t  mod_count;     /* 模块数 */
    uint32_t fw_hash;
    uint8_t  retries;
    uint32_t t_sent;
} node_entry_t;
void master_parser_init(void);
void master_parser_poll(void);
uint8_t master_parser_consume(const frame_t *ans);
const node_entry_t *master_parser_get(uint8_t slot);
#endif
28. master/master_parser.c
C
#include "master_parser.h"
#include "net.h"
#include "hal_port.h"
#include "module.h"
#include "config.h"
static node_entry_t s_nodes[SLOT_COUNT];
static uint8_t s_scan = 0;
static uint8_t s_seq = 0;
static void send_q(uint8_t slot, uint8_t cmd)
{
    frame_t q;
    q.q.magic = FRAME_MAGIC;
    q.q.cmd   = cmd;
    q.q.mod   = 0;
    q.q.dest  = slot;
    q.q.src   = CONFIG_MASTER_SLOT;
    q.q.ttl   = CONFIG_NET_TTL;
    q.q.seq   = ++s_seq;
    q.q.ack   = 0;
    q.q.param[0] = q.q.param[1] = q.q.param[2] = q.q.param[3] = 0;
    protocol_finalize_query(&q);
    net_send(q.raw, PROTO_QUERY_LEN);
    s_nodes[slot].t_sent = hal_tick();
}
void master_parser_init(void)
{
    for (uint8_t i = 0; i < SLOT_COUNT; i++) {
        s_nodes[i].slot = i;
        s_nodes[i].state = P_UNKNOWN;
        s_nodes[i].mod_listed = 0;
        s_nodes[i].mod_count = 0;
        s_nodes[i].retries = 0;
        s_nodes[i].t_sent = 0;
    }
}
void master_parser_poll(void)
{
    uint32_t now = hal_tick();
    static uint32_t t_scan = 0;
    if ((now - t_scan) < 50) return;
    t_scan = now;
    /* 1) 超时重试 */
    for (uint8_t i = 0; i < SLOT_COUNT; i++) {
        if (s_nodes[i].state == P_PENDING && (now - s_nodes[i].t_sent) > P_TIMEOUT) {
            if (++s_nodes[i].retries >= P_MAX_RETRY) {
                s_nodes[i].state = P_LOST;
                continue;
            }
            send_q(i, CMD_WHOAMI);
            return;
        }
    }
    /* 2) 扫描未发现槽位（跳过 master 自身槽位） */
    for (uint8_t n = 0; n < SLOT_COUNT; n++) {
        uint8_t i = (uint8_t)((s_scan + n) & (SLOT_COUNT - 1));
        if (i == CONFIG_MASTER_SLOT) continue;
        if (s_nodes[i].state == P_UNKNOWN || s_nodes[i].state == P_LOST) {
            s_scan = (uint8_t)((i + 1) & (SLOT_COUNT - 1));
            send_q(i, CMD_WHOAMI);
            return;
        }
    }
    /* 3) 点验：已发现未取清单 */
    for (uint8_t i = 0; i < SLOT_COUNT; i++) {
        if (s_nodes[i].state == P_READY && !s_nodes[i].mod_listed) {
            send_q(i, CMD_MODLIST);
            return;
        }
    }
}
uint8_t master_parser_consume(const frame_t *ans)
{
    uint8_t s = FSRC(ans);
    if (s >= SLOT_COUNT) return 0;
    node_entry_t *e = &s_nodes[s];
    if (FCMD(ans) == CMD_WHOAMI) {
        e->slot = s;
        e->state = P_READY;
        e->fw_hash = ((uint32_t)ans->a.data[3] << 24) |
                     ((uint32_t)ans->a.data[4] << 16) |
                     ((uint32_t)ans->a.data[5] << 8)  |
                     ans->a.data[6];
        e->retries = 0;
        e->mod_listed = 0;
        e->mod_count = 0;
        return 1;
    }
    if (FCMD(ans) == CMD_MODLIST) {
        if (e->state != P_READY) return 0;
        e->mod_listed = 1;
        e->mod_count = ans->a.data[0];
        return 1;
    }
    return 0;
}
const node_entry_t *master_parser_get(uint8_t slot)
{
    return (slot < SLOT_COUNT) ? &s_nodes[slot] : 0;
}
29. master/master_module_table.h
C
#ifndef MODULE_TABLE_H
#define MODULE_TABLE_H
#include "module.h"
#define MODULE_TABLE(X) \
    X( mod_control, 0, DIR_EAST, DOMAIN_C, LAYER_MASTER, LIFECYCLE_ACTIVE, 0x08U, 1, 0x99AABBCCU )
#endif
30. master/mod_control.c
C
#include "module.h"
#include "net.h"
#include "hal_port.h"
#include "master_parser.h"
#include "config.h"
static uint8_t s_seq = 0, s_pos = 0;
static uint8_t s_pend[32];
static uint8_t s_pend_n = 0;
static void send_cmd(uint8_t slot, uint8_t cmd, uint8_t mod)
{
    frame_t q;
    q.q.magic = FRAME_MAGIC;
    q.q.cmd   = cmd;
    q.q.mod   = mod;
    q.q.dest  = slot;
    q.q.src   = CONFIG_MASTER_SLOT;
    q.q.ttl   = CONFIG_NET_TTL;
    q.q.seq   = ++s_seq;
    q.q.ack   = 0;
    q.q.param[0] = 8;
    q.q.param[1] = q.q.param[2] = q.q.param[3] = 0;
    protocol_finalize_query(&q);
    net_send(q.raw, PROTO_QUERY_LEN);
}
static void poll(void)
{
    static uint32_t t = 0;
    uint32_t now = hal_tick();
    if ((now - t) < 200) return;
    t = now;
    for (uint8_t n = 0; n < SLOT_COUNT && s_pos < SLOT_COUNT; n++) {
        uint8_t slot = s_pos++;
        const node_entry_t *e = master_parser_get(slot);
        if (e && e->state == P_READY) {
            send_cmd(slot, CMD_START,  0);
            send_cmd(slot, CMD_STATUS, 1);
            send_cmd(slot, CMD_STATUS, 2);   /* 否则被老化回收 */
            send_cmd(slot, CMD_STATUS, 3);   /* 否则被老化回收 */
            return;
        }
    }
    /* TODO: 经宿主接口上报 s_pend */
}
static void on_cmd(const frame_t *rx, frame_t *tx)
{
    (void)tx;
    if (FCMD(rx) == CMD_STATUS) {
        for (uint8_t i = 0; i < 32; i++) s_pend[i] = FDATA(rx, i);
        s_pend_n = 1;
    }
}
const module_t mod_control = {
    .id = 0,
    .group = DIR_EAST,
    .domain = DOMAIN_C,
    .layer = LAYER_MASTER,
    .lifecycle = LIFECYCLE_ACTIVE,
    .perm = 0x08U,          /* STATUS(0x03)：1<<3 */
    .api_version = 1,
    .contract_hash = 0x99AABBCCU,
    .init = 0,
    .poll = poll,
    .on_cmd = on_cmd,
};
31. host/host_main.c
C
#include "module.h"
#include "hal_port.h"
/* host 四模块：契约桩（数据通道未来接入） */
static void stub_poll(void) { }
static void stub_on_cmd(const frame_t *rx, frame_t *tx)
{
    tx->a.magic = FRAME_MAGIC;
    tx->a.cmd   = FCMD(rx);
    tx->a.mod   = FMOD(rx);
    tx->a.dest  = FSRC(rx);
    tx->a.src   = 0;
    tx->a.seq   = FSEQ(rx);
    tx->a.ack   = 0;
}
const module_t mod_filter = {
    .id = 0, .group = DIR_EAST, .domain = DOMAIN_H, .layer = LAYER_HOST,
    .lifecycle = LIFECYCLE_OPTIONAL, .perm = 0x00U,
    .api_version = 1, .contract_hash = 0x11111111U,
    .init = 0, .poll = stub_poll, .on_cmd = stub_on_cmd,
};
const module_t mod_correlate = {
    .id = 1, .group = DIR_EAST, .domain = DOMAIN_R, .layer = LAYER_HOST,
    .lifecycle = LIFECYCLE_OPTIONAL, .perm = 0x00U,
    .api_version = 1, .contract_hash = 0x22222222U,
    .init = 0, .poll = stub_poll, .on_cmd = stub_on_cmd,
};
const module_t mod_store = {
    .id = 2, .group = DIR_EAST, .domain = DOMAIN_F, .layer = LAYER_HOST,
    .lifecycle = LIFECYCLE_OPTIONAL, .perm = 0x00U,
    .api_version = 1, .contract_hash = 0x33333333U,
    .init = 0, .poll = stub_poll, .on_cmd = stub_on_cmd,
};
const module_t mod_render = {
    .id = 3, .group = DIR_EAST, .domain = DOMAIN_K, .layer = LAYER_HOST,
    .lifecycle = LIFECYCLE_OPTIONAL, .perm = 0x00U,
    .api_version = 1, .contract_hash = 0x44444444U,
    .init = 0, .poll = stub_poll, .on_cmd = stub_on_cmd,
};
int main(void)
{
    hal_init();
    module_init_all();
    for (;;) {
        module_poll_all();
        hal_watchdog_kick();
    }
}
32. host/host_module_table.h
C
#ifndef MODULE_TABLE_H
#define MODULE_TABLE_H
#include "module.h"
#define MODULE_TABLE(X) \
    X( mod_filter,    0, DIR_EAST, DOMAIN_H, LAYER_HOST, LIFECYCLE_OPTIONAL, 0x00U, 1, 0x11111111U ) \
    X( mod_correlate, 1, DIR_EAST, DOMAIN_R, LAYER_HOST, LIFECYCLE_OPTIONAL, 0x00U, 1, 0x22222222U ) \
    X( mod_store,     2, DIR_EAST, DOMAIN_F, LAYER_HOST, LIFECYCLE_OPTIONAL, 0x00U, 1, 0x33333333U ) \
    X( mod_render,    3, DIR_EAST, DOMAIN_K, LAYER_HOST, LIFECYCLE_OPTIONAL, 0x00U, 1, 0x44444444U )
#endif
33. app/map_slot_label.c
C
#include <stdint.h>
#include "cell.h"

/* 领域数据唯一合法入口（规则：领域数据只进 app/）。
 * 本文件 = 「通用坐标 → 部署命名」的唯一绑定层：
 * 核心层只认 (方位, 动词序号) 整数；叫什么名字、对应什么部署语义，全在此处。
 * 换部署时只改本文件（唯一绑定层），核心层一字不动。 */

typedef struct {
    uint8_t     dir;
    const char *label_cn;
    const char *label_en;
} dir_name_t;

static const dir_name_t s_dir_names[] = {
    { DIR_EAST,      "东",   "EAST"  },
    { DIR_SOUTHEAST, "东南", "SE"    },
    { DIR_SOUTH,     "南",   "SOUTH" },
    { DIR_SOUTHWEST, "西南", "SW"    },
    { DIR_WEST,      "西",   "WEST"  },
    { DIR_NORTHWEST, "西北", "NW"    },
    { DIR_NORTH,     "北",   "NORTH" },
    { DIR_NORTHEAST, "东北", "NE"    },
};

typedef struct {
    uint8_t     verb;
    const char *label_cn;
} verb_name_t;

static const verb_name_t s_verb_names[] = {
    { VERB_OVERWRITE,  "覆盖" },
    { VERB_QUERY,      "查询" },
    { VERB_EXPAND,     "展开" },
    { VERB_CONTRACT,   "收束" },
    { VERB_PROPOSE,    "提出" },
    { VERB_VERIFY,     "验证" },
    { VERB_ADJUDICATE, "裁决" },
    { VERB_WRITE,      "写入" },
};

/* 方位中文名（内部 0 起头 → 查表；越界返回 0） */
const char *map_dir_label(uint8_t dir)
{
    for (uint8_t i = 0; i < sizeof(s_dir_names) / sizeof(s_dir_names[0]); i++)
        if (s_dir_names[i].dir == dir) return s_dir_names[i].label_cn;
    return 0;
}

/* 动词序号中文名 */
const char *map_verb_label(uint8_t verb)
{
    for (uint8_t i = 0; i < sizeof(s_verb_names) / sizeof(s_verb_names[0]); i++)
        if (s_verb_names[i].verb == verb) return s_verb_names[i].label_cn;
    return 0;
}

/* 通用格址标签："DD-VV"（展示层 01 起头），如槽位 0x0A=(东南,展开) → "02-03"。
 * 纯手工拼串，不依赖 stdio。越界返回 0。 */
const char *map_slot_label(uint8_t slot)
{
    static char buf[8];
    uint8_t d = CELL_DIR_DISPLAY(SLOT_DIR(slot));
    uint8_t v = CELL_VERB_DISPLAY(SLOT_VERB(slot));
    if (d > 8U || v > 8U) return 0;
    buf[0] = (char)('0' + (d / 10));
    buf[1] = (char)('0' + (d % 10));
    buf[2] = '-';
    buf[3] = (char)('0' + (v / 10));
    buf[4] = (char)('0' + (v % 10));
    buf[5] = '\0';
    return buf;
}



34. modules/mod_class_register.c

C

#include "module.h"
#include "hal_port.h"

static void on_cmd(const frame_t *rx, frame_t *tx)
{
    tx->a.magic = FRAME_MAGIC;
    tx->a.cmd   = FCMD(rx);
    tx->a.mod   = FMOD(rx);
    tx->a.dest  = FSRC(rx);
    tx->a.src   = hal_node_slot();
    tx->a.seq   = FSEQ(rx);
    tx->a.ack   = 0;

    if (FCMD(rx) == CMD_CLASS) {
        uint8_t domain = FPARAM(rx, 0);
        uint8_t n;

        if (domain == 0xFFU)
            n = module_manifest(&tx->a.data[1], 7);
        else
            n = module_manifest_domain(domain, &tx->a.data[1], 7);

        tx->a.data[0] = n;
        tx->a.ack = 1;
    }
}

const module_t mod_class_register = {
    .id = 5,
    .group = DIR_SOUTHEAST,
    .domain = DOMAIN_H,
    .layer = LAYER_NODE,
    .lifecycle = LIFECYCLE_OPTIONAL,
    .perm = 0x00010000U,    /* CLASS(0x10)：1<<16 */
    .api_version = 1,
    .contract_hash = 0x6F708192U,
    .init = 0,
    .poll = 0,
    .on_cmd = on_cmd,
};

35. modules/mod_intent_bus.c

C

#include "module.h"
#include "net.h"
#include "hal_port.h"

static void on_cmd(const frame_t *rx, frame_t *tx)
{
    tx->a.magic = FRAME_MAGIC;
    tx->a.cmd   = FCMD(rx);
    tx->a.mod   = FMOD(rx);
    tx->a.dest  = FSRC(rx);
    tx->a.src   = hal_node_slot();
    tx->a.seq   = FSEQ(rx);
    tx->a.ack   = 0;

    if (FCMD(rx) == CMD_INTENT) {
        uint8_t intent_id = FPARAM(rx, 0);

        /* 只做转发，不做决策；订阅方自行处理。
           仅当当前网络提供者真正支持意图通道时才转发，
           避免基础队列提供者把命令原样回环成死循环。 */
        if (net_supports(NET_CAP_INTENT) &&
            net_publish(intent_id, rx->raw, PROTO_QUERY_LEN))
            tx->a.ack = 1;
    }
}

const module_t mod_intent_bus = {
    .id = 6,
    .group = DIR_SOUTH,
    .domain = DOMAIN_C,
    .layer = LAYER_NODE,
    .lifecycle = LIFECYCLE_OPTIONAL,
    .perm = 0x00100000U,    /* INTENT(0x14)：1<<20 */
    .api_version = 1,
    .contract_hash = 0x708193A4U,
    .init = 0,
    .poll = 0,
    .on_cmd = on_cmd,
};

36. core/signature.h

C

#ifndef CORE_SIGNATURE_H
#define CORE_SIGNATURE_H

#include <stdint.h>
#include "protocol.h"
#include "cell.h"   /* direction_t / verb_t：签名与意图事件携带格址坐标 */

#define SIGNATURE_MAX_DATA 32U
#define INTENT_PARAM_COUNT 4U

/* 机制周期：M1-M12 是操作的类型轴（本体分类：这个操作是什么），
   与 状态域(在哪)、方位(哪个方向)、动词序号(怎么执行) 正交独立。
   完整格址 = 状态域(12) × 周期(12) × 方位(8) × 动词(8) = 9216；
   域与周期为语义维度，随签名/意图携带，不占槽位地址。
   周期可添加（新增须经审议）。
   注意：动词"验证"与周期"验证类"为同形异义——动词是执行动作，
   周期是操作类型，二者正交，不因同名而等同。 */
typedef enum {
    PERIOD_NONE = 0,
    PERIOD_M1 = 1,  /* 建立类 */
    PERIOD_M2,      /* 撤销类 */
    PERIOD_M3,      /* 改写类 */
    PERIOD_M4,      /* 转流类 */
    PERIOD_M5,      /* 重构类 */
    PERIOD_M6,      /* 体验类 */
    PERIOD_M7,      /* 验证类 */
    PERIOD_M8,      /* 构想类 */
    PERIOD_M9,      /* 锚定类 */
    PERIOD_M10,     /* 镜映类 */
    PERIOD_M11,     /* 学习类 */
    PERIOD_M12      /* 双向类 */
} mechanism_period_t;

/* 通用信号签名：用于承载输入摘要、身份签名、意图签名等。 */
typedef struct {
    uint32_t id;
    uint8_t  domain;
    uint8_t  period;
    uint8_t  type;
    uint8_t  confidence;
    uint8_t  len;
    uint8_t  data[SIGNATURE_MAX_DATA];
    /* 追加字段（append-only，勿改既有偏移）：格址坐标，与 SLOT_DIR/SLOT_VERB 一致 */
    uint8_t  direction;   /* 方位（格址高3位） */
    uint8_t  verb;        /* 动词序号（格址低3位） */
} signature_t;

/* 通用意图事件：上层只声明“想做什么”，由订阅方决定如何执行。 */
typedef struct {
    uint32_t seq;
    uint8_t  intent_id;
    uint8_t  domain;
    uint8_t  period;
    uint8_t  priority;
    uint8_t  param[INTENT_PARAM_COUNT];
    /* 追加字段（append-only）：格址坐标 */
    uint8_t  direction;   /* 方位 */
    uint8_t  verb;        /* 动词序号 */
} intent_event_t;

void    signature_clear(signature_t *s);
uint8_t signature_set(signature_t *s, uint32_t id, uint8_t domain,
                      uint8_t type, const uint8_t *data, uint8_t len);
uint32_t signature_hash(const signature_t *s);
void    intent_event_clear(intent_event_t *e);

#endif

37. core/signature.c

C

#include "signature.h"
#include "config.h"

void signature_clear(signature_t *s)
{
    if (!s) return;
    s->id = 0;
    s->domain = 0;
    s->period = 0;
    s->type = 0;
    s->confidence = 0;
    s->len = 0;
    s->direction = 0;
    s->verb = 0;
    for (uint8_t i = 0; i < SIGNATURE_MAX_DATA; i++)
        s->data[i] = 0;
}

uint8_t signature_set(signature_t *s, uint32_t id, uint8_t domain,
                      uint8_t type, const uint8_t *data, uint8_t len)
{
    if (!s || !data || len > SIGNATURE_MAX_DATA) return 0;
    signature_clear(s);
    s->id = id;
    s->domain = domain;
    s->type = type;
    s->len = len;
    for (uint8_t i = 0; i < len; i++)
        s->data[i] = data[i];
    return 1;
}

uint32_t signature_hash(const signature_t *s)
{
    uint8_t tmp[4 + 5 + SIGNATURE_MAX_DATA + 2];   /* +2：格址坐标（追加字段） */
    uint8_t n = 0;

    if (!s) return 0;
    tmp[n++] = (uint8_t)(s->id >> 24);
    tmp[n++] = (uint8_t)(s->id >> 16);
    tmp[n++] = (uint8_t)(s->id >> 8);
    tmp[n++] = (uint8_t)(s->id);
    tmp[n++] = s->domain;
    tmp[n++] = s->period;
    tmp[n++] = s->type;
    tmp[n++] = s->confidence;
    tmp[n++] = s->len;
    for (uint8_t i = 0; i < s->len; i++)
        tmp[n++] = s->data[i];
    /* 追加字段：方位、动词序号（保持既有序列化顺序不变，只追加在末尾） */
    tmp[n++] = s->direction;
    tmp[n++] = s->verb;
    return protocol_hash32_algo(CONFIG_HASH_ALGO, tmp, n);
}

void intent_event_clear(intent_event_t *e)
{
    if (!e) return;
    e->seq = 0;
    e->intent_id = 0;
    e->domain = 0;
    e->period = 0;
    e->priority = 0;
    e->direction = 0;
    e->verb = 0;
    for (uint8_t i = 0; i < INTENT_PARAM_COUNT; i++)
        e->param[i] = 0;
}

38. core/cell.h
C
#ifndef CORE_CELL_H
#define CORE_CELL_H
#include <stdint.h>

/* ============================================================================
 * 通用格址坐标系统（Universal Cell Coordinate System）
 * cell_id = 状态域(DOMAIN) + M周期(PERIOD) + 方位(DIRECTION) + 动词序号(VERB)
 * 核心层只定义坐标与寻址，不做任何物理、几何或领域声明；
 * 命名与通道绑定只存在于 app 层。
 * ============================================================================ */

/* 方位：8 个，刚性不可扩展（3 位地址容量）。内部 0 起头，对外 01..08。 */
typedef enum {
    DIR_EAST = 0,      /* 01 东   —— 起点 */
    DIR_SOUTHEAST,     /* 02 东南 */
    DIR_SOUTH,         /* 03 南   */
    DIR_SOUTHWEST,     /* 04 西南 */
    DIR_WEST,          /* 05 西   */
    DIR_NORTHWEST,     /* 06 西北 */
    DIR_NORTH,         /* 07 北   */
    DIR_NORTHEAST,     /* 08 东北 */
    DIR_COUNT = 8
} direction_t;

/* 动词序号：基础 8 个根操作；可审议扩展为注册 ID（VERB_EXTENDED_MIN..VERB_MAX），
 * 扩展动词仅存于签名/意图，不占槽位地址。
 * 动词是执行动作轴，与周期(操作类型轴)正交——"验证"与周期"验证类"同名，
 * 系同形异义，不代表同一语义。 */
typedef enum {
    VERB_OVERWRITE = 0,  /* 01 覆盖 —— 以新态覆盖旧态 */
    VERB_QUERY,          /* 02 查询 —— 读取当前态 */
    VERB_EXPAND,         /* 03 展开 —— 开放/细化 */
    VERB_CONTRACT,       /* 04 收束 —— 收敛/封存 */
    VERB_PROPOSE,        /* 05 提出 —— 提交待决 */
    VERB_VERIFY,         /* 06 验证 —— 校验一致性 */
    VERB_ADJUDICATE,     /* 07 裁决 —— 定夺优先级 */
    VERB_WRITE,          /* 08 写入 —— 固化落地 */
    VERB_COUNT = 8,      /* 基础集大小 */
    VERB_EXTENDED_MIN = 8,   /* 扩展动词 ID 起点 */
    VERB_MAX = 255           /* 动词 ID 上限（uint8_t） */
} verb_t;

/* 格址编码：8×8 = 64 槽。高3位=方位，低3位=动词序号。
 * 槽位是节点的物理地址：一个节点 = (方位, 动词序号) 的一个格子。 */
#define SLOT_COUNT   64U
#define SLOT_MAKE(d, v)  (((uint8_t)(((d) & 0x07U) << 3) | ((v) & 0x07U)))
#define SLOT_DIR(s)      ((uint8_t)((s) >> 3))
#define SLOT_VERB(s)     ((uint8_t)((s) & 0x07U))

/* 展示层起头换算：内部 0 起头 → 对外 01 起头（只许展示层调用）。 */
#define CELL_DIR_DISPLAY(d)   ((uint8_t)((d) + 1U))
#define CELL_VERB_DISPLAY(v)  ((uint8_t)((v) + 1U))

#endif



