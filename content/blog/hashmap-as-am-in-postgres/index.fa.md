---
date: 2026-03-25T21:23:57+03:30
draft: true
title: هش‌مپ به عنوان استوریج در پسگره
summary: تو این پست سعی می‌کنیم یه اکسس متد برای پسگره بنویسیم و روش دیتا ذخیره و بازیابی کنیم.
categories:
  - Internal of Things
  - Database
tags:
  - postgres
  - c
ShowToc: true
TocOpen: false
---
# مقدمه
قبل از عید بود که احسان (یادم باشه لینک لینکدینش رو بگیرم و لینک کنم) باهام در مورد TimescaleDB حرف زد و اونجا بود که فهمیدم پسگره هم داشتن Storage Engineهای متفاوت رو ساپورت می‌کنه. قبل از اون از این نظر MySQL برام شاهکار معماری بود با اون حد زیبا از SoC در این زمینه. رفتم و سرچ کردم و سورس پسگره رو نگاهی انداختم و فهمیدم حق هم داشتم البته. پسگره از یه نسخه‌ای به بعد که الان یادم نیست چند بود، این قابلیت رو اضافه کرده. این شد که امروز، در روز نمی‌دونم چندم جنگ به این فکر افتادم که برا تفریح یه ام (مخفف access method) برای پسگره با استفاده از سورسش و کتاب [PostgreSQL 14 Internals](files/postgresql_internals-14_en.pdf) از Egor Rogov بنویسم. کار باحالی به نظر میاد و فکر کنم قراره چیزای باحالی یاد بگیریم. قبل از اون یه هشت‌مپ می‌نویسیم داخل سی و بعد میریم سر وقت خود ام.
# هش‌مپ
## ساختار
سی، مپ یا دیکشنری یا هر چیزی از این دست نداره و برای اینکه بتونیم داده‌هامون رو ذخیره کنیم و با سرعت نسبتاً خوبی هم بازیابی کنیم، به نظرم هش‌مپ می‌تونه گزینه‌ی خوبی باشه. هش‌مپ هیچ چیز خاصی نداره، یه تابع هش داریم و یه آرایه از لیست‌های پیوندی. هر وقتی که یچی (بر مبنای یه کلید) رو قراره ذخیره کنیم، اون کلید رو هش می‌کنیم، بسته به تابع هش اگه نیاز باشه باقیمانده‌ی حاصل رو بر طول آرایه بدست میاریم و المان رو به لیست پیوندی اون ایندکس اضافه می‌کنیم.

بازیابی و حذف هم طبعاً از همین مسیر می‌گذرن. کلید رو هش می‌کنیم، ایندکس آرایه رو پیدا می‌کنیم و لیست رو تا پیدا کردن/نکردن المانی که می‌خوایم بخونیم/حذف کنیم پیمایش می‌کنیم. ایزی.

و اینکه فعلن برای ساده شدن مسئله، فرض می‌کنیم قراره فقط یه عدد صحیح ذخیره کنیم. بعد بر می‌گردیم تاپلش می‌کنیم.
## لیست پیوندی
لیست پیوندی که چیزی نداره. هر المان یه کلید و مقدار داره و برای لیستی که میخوایم بهش اضافه/حذف/پیدا کنیم، از هد میریم و المان به المان کلیدی که می‌خوایم رو چک می‌کنیم و طبعاً اگه نباشه یا به آخر لیست اضافه می‌کنیم (برای اضافه کردن)، اگه باشه آپدیت/حذف می‌کنیم و یا بی‌خیال میشیم (برای حذف/یافتن).

هدر لیست پیوندی‌مون:
```c
#pragma once  
  
#include <stdbool.h>  
  
typedef struct LinkedListData *LinkedList;  
  
struct LinkedListData {  
    int key;  
    int value;  
    LinkedList next;  
};  
  
LinkedList create_linkedlist_node(int key, int value);  
  
LinkedList insert_linkedlist(LinkedList ll, int key, int value);  
  
LinkedList find_linkedlist(LinkedList ll, int key);  
  
bool delete_linkedlist(LinkedList ll, int key);
```
و کدش:
```c
#include <stddef.h>  
#include <stdlib.h>  
#include "linkedlist.h"  
  
LinkedList create_linkedlist_node(const int key, const int value) {  
    LinkedList ll = malloc(sizeof(struct LinkedListData));  
    ll->key = key;  
    ll->value = value;  
    ll->next = NULL;  
  
    return ll;  
}  
  
LinkedList insert_linkedlist(LinkedList ll, const int key, const int value) {  
    LinkedList current;  
    for (current = ll; current != NULL; current = current->next) {  
        if (current->key == key) {  
            current->value = value;  
            return current;  
        }  
        if (current->next == NULL)  
            break;  
    }  
  
    LinkedList new_node = create_linkedlist_node(key, value);  
    if (current == NULL) {  
        *ll = *new_node;  
    } else {  
        current->next = new_node;  
    }  
  
    return new_node;  
}  
  
LinkedList find_linkedlist(LinkedList ll, const int key) {  
    for (LinkedList current = ll; current != NULL; current = current->next) {  
        if (ll->key == key)  
            return current;  
    }  
  
    return NULL;  
}  
  
bool delete_linkedlist(LinkedList ll, const int key) {  
    for (LinkedList current = ll, prev = ll; current != NULL; prev = current, current = current->next) {  
        if (current->key == key) {  
            prev->next = current->next;  
            free(current);  
            return true;  
        }  
    }  
  
    return false;  
}
```
## کد هش‌مپ
و با داشتن لیست پیوندی، همونطور که اول کار اومد، هش‌مپ چیز خاصی دیگه نیست ماجراش. کد هدرمون:
```c
#pragma once  
  
#include <stdbool.h>  
  
#define MAX_SIZE 100  
  
LinkedList hashmap[MAX_SIZE];  
  
int hash(int key);  
  
void insert_hashmap(int key, int value);  
  
bool lookup_hashmap(int key, int *value);  
  
int delete_hashmap(int key);
```
و کد پیاده‌سازیش:
```c
#include <stddef.h>  
#include "linkedlist.h"  
#include "hashmap.h"  
  
int hash(const int key) {  
    return key % MAX_SIZE;  
}  
  
void insert_hashmap(int key, int value) {  
    LinkedList ll = hashmap[hash(key)];  
    if (ll == NULL) {  
        LinkedList node = create_linkedlist_node(key, value);  
        hashmap[hash(key)] = node;  
        return;  
    }  
  
    insert_linkedlist(hashmap[hash(key)], key, value);  
}

bool contain_hashmap(int key) {
	LinkedList ll = find_linkedlist(hashmap[hash(key)], key);  
    
	return ll != NULL;
}
  
bool lookup_hashmap(int key, int *value) {  
    LinkedList ll = find_linkedlist(hashmap[hash(key)], key);  
    if (ll != NULL) {  
        *value = ll->value;  
        return true;  
    }  
    return false;  
}  
  
int delete_hashmap(int key) {  
    return delete_linkedlist(hashmap[hash(key)], key);  
}
```
خوبه. الان رسیدیم به جایی که می‌تونیم بپریم تو سورس پسگره.
# سرآیندهای توسعه‌ی پسگره
برای ادامه‌ی مسیر به حضور فایل‌های توسعه‌ی پسگره نیاز داریم. با نصب پکیج‌هایی مثل `postgresql-devel` یا اینطوری چیزی میشه این سرآیندها رو به سیستم اضافه کرد ولی خب از اونجا که قراره با سورس بریم جلو، میشه پسگره رو از `git@github.com:postgres/postgres.git` کلون کرد و با کامندهای زیر کامپایل و نصبش کرد:
```
$ export PKG_CONFIG_PATH="$(brew --prefix icu4c)/lib/pkgconfig:$PKG_CONFIG_PATH"
$ ./configure --enable-cassert --enable-debug CFLAGS="-O0 -g3 -fno-omit-frame-pointer"
$ make clean && make -j 8 && sudo make install
```
احتمالاً به کانفیگ `icu4c` داخل لینوکس نیازی نباشه و طبعاً خط اول نیازی نیست. بعد از نصب، یه نسخه پسگره در مسیر `/usr/local/pgsql` وجود داره.
# اکسس‌متد
اکسس‌متدها (و یا به اختصار که قرار شد بهشون بگیم ام) داخل پسگره دو نوع دارن: جدول‌ها و ایندکس‌ها. ما قراره ام جدول رو پیاده کنیم. داخل سورس‌کدی که من از پسگره دارم (مستر روی هش b7057e43467ff2d7c)، از فایل `src/include/access/tableam.h` میشه استراکچری که باید پیاده کنیم رو دید و طبعاً برای تقلب هم میشه سورس هیپ رو در `src/backend/access/heap/heapam_handler.c` دید (در واقع تنها ام جدولی که به صورت دیفالت داخل پسگره هست، همین هیپه). پس میریم برای ساختن `hmam.c` :دی.
## اسکلت ماجرا
اگه فایل ‍`heapam_handler` رو ببینیم، یه سری تابع اول کار نوشته شده تا آخر کار رسیده به یه استراکت `TableAmRoutine` که تابع‌هایی که بالا نوشته شده رو داره اساین می‌کنه به پوینتر فانکشن‌هایی که داخل `TableAmRoutine` هستن. توضیح هر تابع هم به صورت کامل داخل استراکتی به همین اسم، داخل فایل `tableam` اومده. به نظر نقطه‌ی شروع می‌تونه کپی کردن اون فایل داخل فایل سی خودمون باشه:
```c
#include "postgres.h"  
  
static const TableAmRoutine heapam_methods = {  
    .type = T_TableAmRoutine,  
  
    .slot_callbacks = heapam_slot_callbacks,  
  
    .scan_begin = heap_beginscan,  
    .scan_end = heap_endscan,  
    .scan_rescan = heap_rescan,  
    .scan_getnextslot = heap_getnextslot,  
  
    .scan_set_tidrange = heap_set_tidrange,  
    .scan_getnextslot_tidrange = heap_getnextslot_tidrange,  
  
    .parallelscan_estimate = table_block_parallelscan_estimate,  
    .parallelscan_initialize = table_block_parallelscan_initialize,  
    .parallelscan_reinitialize = table_block_parallelscan_reinitialize,  
  
    .index_fetch_begin = heapam_index_fetch_begin,  
    .index_fetch_reset = heapam_index_fetch_reset,  
    .index_fetch_end = heapam_index_fetch_end,  
    .index_fetch_tuple = heapam_index_fetch_tuple,  
  
    .tuple_insert = heapam_tuple_insert,  
    .tuple_insert_speculative = heapam_tuple_insert_speculative,  
    .tuple_complete_speculative = heapam_tuple_complete_speculative,  
    .multi_insert = heap_multi_insert,  
    .tuple_delete = heapam_tuple_delete,  
    .tuple_update = heapam_tuple_update,  
    .tuple_lock = heapam_tuple_lock,  
  
    .tuple_fetch_row_version = heapam_fetch_row_version,  
    .tuple_get_latest_tid = heap_get_latest_tid,  
    .tuple_tid_valid = heapam_tuple_tid_valid,  
    .tuple_satisfies_snapshot = heapam_tuple_satisfies_snapshot,  
    .index_delete_tuples = heap_index_delete_tuples,  
  
    .relation_set_new_filelocator = heapam_relation_set_new_filelocator,  
    .relation_nontransactional_truncate = heapam_relation_nontransactional_truncate,  
    .relation_copy_data = heapam_relation_copy_data,  
    .relation_copy_for_cluster = heapam_relation_copy_for_cluster,  
    .relation_vacuum = heap_vacuum_rel,  
    .scan_analyze_next_block = heapam_scan_analyze_next_block,  
    .scan_analyze_next_tuple = heapam_scan_analyze_next_tuple,  
    .index_build_range_scan = heapam_index_build_range_scan,  
    .index_validate_scan = heapam_index_validate_scan,  
  
    .relation_size = table_block_relation_size,  
    .relation_needs_toast_table = heapam_relation_needs_toast_table,  
    .relation_toast_am = heapam_relation_toast_am,  
    .relation_fetch_toast_slice = heap_fetch_toast_slice,  
  
    .relation_estimate_size = heapam_estimate_rel_size,  
  
    .scan_bitmap_next_tuple = heapam_scan_bitmap_next_tuple,  
    .scan_sample_next_block = heapam_scan_sample_next_block,  
    .scan_sample_next_tuple = heapam_scan_sample_next_tuple  
};
```
اگه از یه IDE برای توسعه استفاده کنیم (من از Clion استفاده می‌کنم) می‌بینیم که ادیتور هیچ خطایی نمی‌گیره و حتی سرآیند `postgres.h` رو هم نمی‌شناسه. این یعنی نتونسته مسیرها رو درست بفهمه. برای فیکس این مشکل، ما نیاز به یه بیلدسیستم داریم که خود پسگره از `make` استفاده می‌کنه.
## ابزار `make`
برای `make` کافیه فایل `Makefile` رو بسازیم. ولی چی بذاریم توش؟ اگه داخل سورس پسگره بچرخیم، فایل‌های `Makefile` زیادی میشه دید که به نظر نمی‌رسه کمک کنن. با یذره بیشتر چرخیدن به فولدر `src/makefiles` می‌رسیم که اگه توش بچرخیم میشه فایل `src/makefiles/pgxs.mk` رو دید. این فایل میک‌فایل برا ساختن اکستنشن برای پسگره‌س. کامنت‌هاش به صورت کامل گفته باید چیکار کرد، پس یه اینطور چیزی میشه `Makefile`مون:
```Makefile
MODULES = hmam
EXTENSION = hmam  
  
OBJS = hashmap.o linkedlist.o  
  
PG_CONFIG = /usr/local/pgsql/bin/pg_config  
PGXS := $(shell $(PG_CONFIG) --pgxs)  
include $(PGXS)
```
دقت کنیم که مسیر `PG_CONFIG` چیزیه که بالا بعد از کامپایل پسگره بدست آوردیم. و با استفاده از `EXTENSION` باید یه فایل به اسمی که دادیم و پسوند `control` هم داخل سورس‌مون بسازیم (یعنی `hmam.control`). الان اگه `make` بگیریم به خطا می‌خوریم به خاطر اینکه محتویات هیپ رو کپی کردیم داخل فایل خودمون و خب این خوبه.
## کامپایل اول
برای ادامه پلن اینه:
- دونه دونه روی اسم فیلدهای استراکت `TableAmRoutine` کلیک می‌کنیم
- سیگنیچر پوینتر به فانکشن فیلد رو کپی می‌کنیم
- سیگنیچر رو میاریم به صورت استاتیک می‌ذاریم داخل فایل خودمون
- پرانتز و ستاره و پرانتز بسته‌ی دور اسم رو بر میداریم و یه `hmam_` به اولش اضافه می‌کنیم
- اگه فانکشن باید چیزی برگردونه اون رو می‌ذاریم مقدار صفر تایپ
- اسم تابع جدیدمون رو می‌ذاریم جلوی فیلد

پس فایل‌مون تبدیل میشه به این (و این آخرین باریه که فایل کامل رو داخل پست می‌بینیم :دی):
```c
#include <stddef.h>  
#include "postgres.h"  
#include "access/tableam.h"  
  
static const TupleTableSlotOps *  
hmam_slot_callbacks(Relation rel) {  
    return NULL;  
}  
  
static TableScanDesc  
hmam_scan_begin(Relation rel,  
                Snapshot snapshot,  
                int nkeys, ScanKeyData *key,  
                ParallelTableScanDesc pscan,  
                uint32 flags) {  
    return NULL;  
}  
  
static void  
hmam_scan_end(TableScanDesc scan) {  
}  
  
static void  
hmam_scan_rescan(TableScanDesc scan, ScanKeyData *key,  
                 bool set_params, bool allow_strat,  
                 bool allow_sync, bool allow_pagemode) {  
}  
  
static bool  
hmam_scan_getnextslot(TableScanDesc scan,  
                      ScanDirection direction,  
                      TupleTableSlot *slot) {  
    return false;  
}  
  
static void  
hmam_scan_set_tidrange(TableScanDesc scan,  
                       ItemPointer mintid,  
                       ItemPointer maxtid) {  
}  
  
static bool  
hmam_scan_getnextslot_tidrange(TableScanDesc scan,  
                               ScanDirection direction,  
                               TupleTableSlot *slot) {  
    return false;  
}  
  
static IndexFetchTableData *  
hmam_index_fetch_begin(Relation rel) {  
    return NULL;  
}  
  
static void  
hmam_index_fetch_reset(IndexFetchTableData *data) {  
}  
  
static void  
hmam_index_fetch_end(IndexFetchTableData *data) {  
}  
  
static bool  
hmam_index_fetch_tuple(IndexFetchTableData *scan,  
                       ItemPointer tid,  
                       Snapshot snapshot,  
                       TupleTableSlot *slot,  
                       bool *call_again, bool *all_dead) {  
    return false;  
}  
  
static void  
hmam_tuple_insert(Relation rel, TupleTableSlot *slot,  
                  CommandId cid, int options,  
                  BulkInsertStateData *bistate) {  
}  
  
static void  
hmam_tuple_insert_speculative(Relation rel,  
                              TupleTableSlot *slot,  
                              CommandId cid,  
                              int options,  
                              BulkInsertStateData *bistate,  
                              uint32 specToken) {  
}  
  
static void  
hmam_tuple_complete_speculative(Relation rel,  
                                TupleTableSlot *slot,  
                                uint32 specToken,  
                                bool succeeded) {  
}  
  
static void  
hmam_multi_insert(Relation rel, TupleTableSlot **slots, int nslots,  
                  CommandId cid, int options, BulkInsertStateData *bistate) {  
}  
  
static TM_Result  
hmam_tuple_delete(Relation rel,  
                  ItemPointer tid,  
                  CommandId cid,  
                  Snapshot snapshot,  
                  Snapshot crosscheck,  
                  bool wait,  
                  TM_FailureData *tmfd,  
                  bool changingPart) {  
    return TM_Ok;  
}  
  
static TM_Result  
hmam_tuple_update(Relation rel,  
                  ItemPointer otid,  
                  TupleTableSlot *slot,  
                  CommandId cid,  
                  Snapshot snapshot,  
                  Snapshot crosscheck,  
                  bool wait,  
                  TM_FailureData *tmfd,  
                  LockTupleMode *lockmode,  
                  TU_UpdateIndexes *update_indexes) {  
    return TM_Ok;  
}  
  
static TM_Result  
hmam_tuple_lock(Relation rel,  
                ItemPointer tid,  
                Snapshot snapshot,  
                TupleTableSlot *slot,  
                CommandId cid,  
                LockTupleMode mode,  
                LockWaitPolicy wait_policy,  
                uint8 flags,  
                TM_FailureData *tmfd) {  
    return TM_Ok;  
}  
  
static bool  
hmam_tuple_fetch_row_version(Relation rel,  
                             ItemPointer tid,  
                             Snapshot snapshot,  
                             TupleTableSlot *slot) {  
    return false;  
}  
  
static void  
hmam_tuple_get_latest_tid(TableScanDesc scan,  
                          ItemPointer tid) {  
}  
  
static bool  
hmam_tuple_tid_valid(TableScanDesc scan,  
                     ItemPointer tid) {  
    return false;  
}  
  
static bool  
hmam_tuple_satisfies_snapshot(Relation rel,  
                              TupleTableSlot *slot,  
                              Snapshot snapshot) {  
    return false;  
}  
  
static TransactionId  
hmam_index_delete_tuples(Relation rel,  
                         TM_IndexDeleteOp *delstate) {  
    return 0;  
}  
  
static void  
hmam_relation_set_new_filelocator(Relation rel,  
                                  const RelFileLocator *newrlocator,  
                                  char persistence,  
                                  TransactionId *freezeXid,  
                                  MultiXactId *minmulti) {  
}  
  
static void  
hmam_relation_nontransactional_truncate(Relation rel) {  
}  
  
static void  
hmam_relation_copy_data(Relation rel,  
                        const RelFileLocator *newrlocator) {  
}  
  
static void  
hmam_relation_copy_for_cluster(Relation OldTable,  
                               Relation NewTable,  
                               Relation OldIndex,  
                               bool use_sort,  
                               TransactionId OldestXmin,  
                               TransactionId *xid_cutoff,  
                               MultiXactId *multi_cutoff,  
                               double *num_tuples,  
                               double *tups_vacuumed,  
                               double *tups_recently_dead) {  
}  
  
static void  
hmam_relation_vacuum(Relation rel,  
                     const VacuumParams params,  
                     BufferAccessStrategy bstrategy) {  
}  
  
static bool  
hmam_scan_analyze_next_block(TableScanDesc scan,  
                             ReadStream *stream) {  
    return false;  
}  
  
static bool  
hmam_scan_analyze_next_tuple(TableScanDesc scan,  
                             TransactionId OldestXmin,  
                             double *liverows,  
                             double *deadrows,  
                             TupleTableSlot *slot) {  
    return false;  
}  
  
static double  
hmam_index_build_range_scan(Relation table_rel,  
                            Relation index_rel,  
                            IndexInfo *index_info,  
                            bool allow_sync,  
                            bool anyvisible,  
                            bool progress,  
                            BlockNumber start_blockno,  
                            BlockNumber numblocks,  
                            IndexBuildCallback callback,  
                            void *callback_state,  
                            TableScanDesc scan) {  
    return 0;  
}  
  
static void  
hmam_index_validate_scan(Relation table_rel,  
                         Relation index_rel,  
                         IndexInfo *index_info,  
                         Snapshot snapshot,  
                         ValidateIndexState *state) {  
}  
  
static bool  
hmam_relation_needs_toast_table(Relation rel) {  
    return false;  
}  
  
static Oid  
hmam_relation_toast_am(Relation rel) {  
    return 0;  
}  
  
static void  
hmam_relation_fetch_toast_slice(Relation toastrel, Oid valueid,  
                                int32 attrsize,  
                                int32 sliceoffset,  
                                int32 slicelength,  
                                struct varlena *result) {  
}  
  
static void  
hmam_relation_estimate_size(Relation rel, int32 *attr_widths,  
                            BlockNumber *pages, double *tuples,  
                            double *allvisfrac) {  
}  
  
static bool  
hmam_scan_bitmap_next_tuple(TableScanDesc scan,  
                            TupleTableSlot *slot,  
                            bool *recheck,  
                            uint64 *lossy_pages,  
                            uint64 *exact_pages) {  
    return false;  
}  
  
static bool  
hmam_scan_sample_next_block(TableScanDesc scan,  
                            SampleScanState *scanstate) {  
    return false;  
}  
  
static bool  
hmam_scan_sample_next_tuple(TableScanDesc scan,  
                            SampleScanState *scanstate,  
                            TupleTableSlot *slot) {  
    return false;  
}  
  
static const TableAmRoutine hmam_methods = {  
    .type = T_TableAmRoutine,  
  
    .slot_callbacks = hmam_slot_callbacks,  
  
    .scan_begin = hmam_scan_begin,  
    .scan_end = hmam_scan_end,  
    .scan_rescan = hmam_scan_rescan,  
    .scan_getnextslot = hmam_scan_getnextslot,  
  
    .scan_set_tidrange = hmam_scan_set_tidrange,  
    .scan_getnextslot_tidrange = hmam_scan_getnextslot_tidrange,  
  
    .parallelscan_estimate = table_block_parallelscan_estimate,  
    .parallelscan_initialize = table_block_parallelscan_initialize,  
    .parallelscan_reinitialize = table_block_parallelscan_reinitialize,  
  
    .index_fetch_begin = hmam_index_fetch_begin,  
    .index_fetch_reset = hmam_index_fetch_reset,  
    .index_fetch_end = hmam_index_fetch_end,  
    .index_fetch_tuple = hmam_index_fetch_tuple,  
  
    .tuple_insert = hmam_tuple_insert,  
    .tuple_insert_speculative = hmam_tuple_insert_speculative,  
    .tuple_complete_speculative = hmam_tuple_complete_speculative,  
    .multi_insert = hmam_multi_insert,  
    .tuple_delete = hmam_tuple_delete,  
    .tuple_update = hmam_tuple_update,  
    .tuple_lock = hmam_tuple_lock,  
  
    .tuple_fetch_row_version = hmam_tuple_fetch_row_version,  
    .tuple_get_latest_tid = hmam_tuple_get_latest_tid,  
    .tuple_tid_valid = hmam_tuple_tid_valid,  
    .tuple_satisfies_snapshot = hmam_tuple_satisfies_snapshot,  
    .index_delete_tuples = hmam_index_delete_tuples,  
  
    .relation_set_new_filelocator = hmam_relation_set_new_filelocator,  
    .relation_nontransactional_truncate = hmam_relation_nontransactional_truncate,  
    .relation_copy_data = hmam_relation_copy_data,  
    .relation_copy_for_cluster = hmam_relation_copy_for_cluster,  
    .relation_vacuum = hmam_relation_vacuum,  
    .scan_analyze_next_block = hmam_scan_analyze_next_block,  
    .scan_analyze_next_tuple = hmam_scan_analyze_next_tuple,  
    .index_build_range_scan = hmam_index_build_range_scan,  
    .index_validate_scan = hmam_index_validate_scan,  
  
    .relation_size = table_block_relation_size,  
    .relation_needs_toast_table = hmam_relation_needs_toast_table,  
    .relation_toast_am = hmam_relation_toast_am,  
    .relation_fetch_toast_slice = hmam_relation_fetch_toast_slice,  
  
    .relation_estimate_size = hmam_relation_estimate_size,  
  
    .scan_bitmap_next_tuple = hmam_scan_bitmap_next_tuple,  
    .scan_sample_next_block = hmam_scan_sample_next_block,  
    .scan_sample_next_tuple = hmam_scan_sample_next_tuple  
};  
  
Datum  
hmam_tableam_handler(PG_FUNCTION_ARGS) {  
    PG_RETURN_POINTER(&hmam_methods);  
}
```
چهار خط آخر هم که اومدیم و هندلری که می‌خوایم بعداُ داخل پسگره ازش استفاده کنیم رو معرفی کردیم و گفتیم که آدرس استراکتی که پر کردیم رو برگردونه. اگه بزنیم تا فایل نصب و بیلد شه به این می‌رسیم:
```shell
meysam@Meysams-Mac pgam % make clean && make && sudo make install
rm -f hmam.dylib hmam.o  \
	    hmam.bc
rm -f hashmap.o linkedlist.o hashmap.bc linkedlist.bc
gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Werror=unguarded-availability-new -Wendif-labels -Wmissing-format-attribute -Wcast-function-type -Wformat-security -Wmissing-variable-declarations -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-unused-command-line-argument -Wno-compound-token-split-by-macro -Wno-format-truncation -Wno-cast-function-type-strict -g -O0 -g3 -fno-omit-frame-pointer  -fvisibility=hidden -I. -I./ -I/usr/local/pgsql/include/server -I/usr/local/pgsql/include/internal -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX26.2.sdk   -I/opt/homebrew/Cellar/icu4c@78/78.1/include    -c -o hmam.o hmam.c
gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Werror=unguarded-availability-new -Wendif-labels -Wmissing-format-attribute -Wcast-function-type -Wformat-security -Wmissing-variable-declarations -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-unused-command-line-argument -Wno-compound-token-split-by-macro -Wno-format-truncation -Wno-cast-function-type-strict -g -O0 -g3 -fno-omit-frame-pointer  -fvisibility=hidden hmam.o -L/usr/local/pgsql/lib -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX26.2.sdk   -Wl,-dead_strip_dylibs   -fvisibility=hidden -bundle -bundle_loader /usr/local/pgsql/bin/postgres -o hmam.dylib
/bin/sh /usr/local/pgsql/lib/pgxs/src/makefiles/../../config/install-sh -c -d '/usr/local/pgsql/share/extension'
/bin/sh /usr/local/pgsql/lib/pgxs/src/makefiles/../../config/install-sh -c -d '/usr/local/pgsql/lib'
/usr/bin/install -c -m 644 .//hmam.control '/usr/local/pgsql/share/extension/'
/usr/bin/install -c -m 755  hmam.dylib '/usr/local/pgsql/lib/'

meysam@Meysams-Mac pgam % ll /usr/local/pgsql/lib/hmam.dylib
-rwxr-xr-x@ 1 root  wheel    38K Mar 26 19:22 /usr/local/pgsql/lib/hmam.dylib
```
## نصب اول
ما اکسس‌متدمون رو کامپایل کردیم و داخل پسگره قرارش دادیم. حالا پسگره‌ای که کامپایل کردیم رو میاریم بالا:
```shell
meysam@Meysams-Mac pgam % /usr/local/pgsql/bin/initdb myhmdb
The files belonging to this database system will be owned by user "meysam".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

creating directory myhmdb ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default "max_connections" ... 100
selecting default "shared_buffers" ... 128MB
selecting default time zone ... Asia/Tehran
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

initdb: warning: enabling "trust" authentication for local connections
initdb: hint: You can change this by editing pg_hba.conf or using the option -A, or --auth-local and --auth-host, the next time you run initdb.

Success. You can now start the database server using:

    /usr/local/pgsql/bin/pg_ctl -D myhmdb -l logfile start

meysam@Meysams-Mac pgam % ll myhmdb
total 128
drwx------@  5 meysam  staff   160B Mar 26 17:26 base
drwx------@ 60 meysam  staff   1.9K Mar 26 17:26 global
drwx------@  2 meysam  staff    64B Mar 26 17:26 pg_commit_ts
drwx------@  2 meysam  staff    64B Mar 26 17:26 pg_dynshmem
-rw-------@  1 meysam  staff   5.6K Mar 26 17:26 pg_hba.conf
-rw-------@  1 meysam  staff   2.6K Mar 26 17:26 pg_ident.conf
drwx------@  5 meysam  staff   160B Mar 26 17:26 pg_logical
drwx------@  4 meysam  staff   128B Mar 26 17:26 pg_multixact
drwx------@  2 meysam  staff    64B Mar 26 17:26 pg_notify
drwx------@  2 meysam  staff    64B Mar 26 17:26 pg_replslot
drwx------@  2 meysam  staff    64B Mar 26 17:26 pg_serial
drwx------@  2 meysam  staff    64B Mar 26 17:26 pg_snapshots
drwx------@  3 meysam  staff    96B Mar 26 17:26 pg_stat
drwx------@  2 meysam  staff    64B Mar 26 17:26 pg_stat_tmp
drwx------@  3 meysam  staff    96B Mar 26 17:26 pg_subtrans
drwx------@  2 meysam  staff    64B Mar 26 17:26 pg_tblspc
drwx------@  2 meysam  staff    64B Mar 26 17:26 pg_twophase
-rw-------@  1 meysam  staff     3B Mar 26 17:26 PG_VERSION
drwx------@  5 meysam  staff   160B Mar 26 17:26 pg_wal
drwx------@  3 meysam  staff    96B Mar 26 17:26 pg_xact
-rw-------@  1 meysam  staff    88B Mar 26 17:26 postgresql.auto.conf
-rw-------@  1 meysam  staff    43K Mar 26 17:26 postgresql.conf
meysam@Meysams-Mac pgam % /usr/local/pgsql/bin/pg_ctl -D myhmdb -l logfile start
waiting for server to start.... done
server started
```
و تلاش می‌کنیم امی که ساختیم رو داخل پسگره نصب کنیم. برای لود کردن اکستنشن به پسگره‌مون وصل میشیم و البته قبلش از جدول `pg_am` لیست ام‌های فعلی رو می‌گیریم:
```sql
postgres=# SELECT * FROM pg_am;
 oid  | amname |      amhandler       | amtype
------+--------+----------------------+--------
    2 | heap   | heap_tableam_handler | t
  403 | btree  | bthandler            | i
  405 | hash   | hashhandler          | i
  783 | gist   | gisthandler          | i
 2742 | gin    | ginhandler           | i
 4000 | spgist | spghandler           | i
 3580 | brin   | brinhandler          | i
(7 rows)
```
پسگره به صورت دیفالت یک ام جدول به اسم `heap` داره و شش تا ام ایندکس. حالا میریم برای نصب کردن اکستنشن‌مون:
```sql
postgres=# CREATE OR REPLACE FUNCTION hmam_table_handler(internal) RETURNS table_am_handler AS 'hmam' LANGUAGE C STRICT;
ERROR:  incompatible library "/usr/local/pgsql/lib/hmam.dylib": missing magic block
HINT:  Extension libraries are required to use the PG_MODULE_MAGIC macro.
```
عالی! خطا گرفتیم که ماکروی `PG_MODULE_MAGIC` حتماً داخل اکستنشن استفاده شه. اگه این ماکرو رو داخل سورس کد پسگره سرچ کنیم به سرآیند `fmgr` می‌رسیم. اضافه‌اش می‌کنیم و ماکرو رو صدا می‌کنیم:
```diff
#include "access/tableam.h"  
+#include "fmgr.h"  
  
+PG_MODULE_MAGIC;  
  
static const TupleTableSlotOps *
```
و دوباره کامپایل و نصب می‌کنیم و کوئری نصب اکستنشن رو لود می‌کنیم:
```sql
postgres=# CREATE OR REPLACE FUNCTION hmam_table_handler(internal) RETURNS table_am_handler AS 'hmam' LANGUAGE C STRICT;
ERROR:  could not find function "hmam_table_handler" in file "/usr/local/pgsql/lib/hmam.dylib"
```
چه خوب. فایل‌مون رو تونسته پیدا کنه، ولی انگار فانکشن اکسپورت نشده. به سورس‌کد پسگره بر می‌گردیم و فایل `fmgr.h` رو می‌خونیم. بعد از خوندن متوجه می‌شیم باید ماکروی `PG_FUNCTION_INFO_V1` رو روی فانکشن هندلرمون اعمال کنیم تا چندتا `extern` برامون بسازه:
```diff
+PG_FUNCTION_INFO_V1(hmam_tableam_handler);  
  
Datum  
hmam_tableam_handler(PG_FUNCTION_ARGS) {  
    PG_RETURN_POINTER(&hmam_methods);  
}
```
کامپایل و نصب و اکستنشن رو لود می‌کنیم:
```sql
postgres=# CREATE OR REPLACE FUNCTION hmam_tableam_handler(internal) RETURNS table_am_handler AS 'hmam' LANGUAGE C STRICT;
CREATE FUNCTION
```
هندلرمون رو تونست پیدا کنه و مونده ثبت کردنش به عنوان ام:
```sql
postgres=# CREATE ACCESS METHOD hmam TYPE TABLE HANDLER hmam_tableam_handler;
CREATE ACCESS METHOD
postgres=# SELECT * FROM pg_am;
  oid  | amname |      amhandler       | amtype
-------+--------+----------------------+--------
     2 | heap   | heap_tableam_handler | t
   403 | btree  | bthandler            | i
   405 | hash   | hashhandler          | i
   783 | gist   | gisthandler          | i
  2742 | gin    | ginhandler           | i
  4000 | spgist | spghandler           | i
  3580 | brin   | brinhandler          | i
 16419 | hmam   | hmam_tableam_handler | t
(8 rows)
```
عالی! ما الان یه ام به اسم `hmam` داریم.
## اولین جدول
الان ما اکسس‌متدمون داخل پسگره تعریف شده. بریم یه جدول باهاش بسازیم:
```sql
postgres=# CREATE TABLE a(b int) using hmam;
CREATE TABLE
```
و ازش بخونیم:
```sql
postgres=# SELECT * FROM a;
server closed the connection unexpectedly
	This probably means the server terminated abnormally
	before or while processing the request.
The connection to the server was lost. Attempting reset: Failed.
The connection to the server was lost. Attempting reset: Failed.
```
اوپس! دیتابیس‌مون ترکید :)).
## اولین خطا
وقتی داشتیم سرور پسگره رو ران می‌کردیم فایل لاگ هم بهش دادیم که تو مسیر کدمونه و اگه چکش کنیم، محتویاتش اینه:
```log
2026-03-26 19:28:59.968 +0330 [40142] LOG:  client backend (PID 40478) was terminated by signal 11: Segmentation fault: 11  
2026-03-26 19:28:59.968 +0330 [40142] DETAIL:  Failed process was running: SELECT * FROM a;  
2026-03-26 19:28:59.968 +0330 [40142] LOG:  terminating any other active server processes  
2026-03-26 19:28:59.976 +0330 [40142] LOG:  all server processes terminated; reinitializing  
2026-03-26 19:28:59.987 +0330 [52859] LOG:  database system was interrupted; last known up at 2026-03-26 19:28:23 +0330  
2026-03-26 19:29:00.039 +0330 [52859] LOG:  database system was not properly shut down; automatic recovery in progress  
2026-03-26 19:29:00.041 +0330 [52859] LOG:  redo starts at 0/018919F0  
2026-03-26 19:29:00.043 +0330 [52859] LOG:  invalid record length at 0/018A58D8: expected at least 24, got 0  
2026-03-26 19:29:00.043 +0330 [52859] LOG:  redo done at 0/018A58A0 system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s  
2026-03-26 19:29:00.044 +0330 [52860] LOG:  checkpoint starting: end-of-recovery fast wait  
2026-03-26 19:29:00.046 +0330 [52860] LOG:  checkpoint complete: wrote 19 buffers (0.1%), wrote 3 SLRU buffers; 0 WAL file(s) added, 0 removed, 0 recycled; write=0.002 s, sync=0.001 s, total=0.003 s; sync files=18, longest=0.001 s, average=0.001 s; distance=79 kB, estimate=79 kB; lsn=0/018A58D8, redo lsn=0/018A58D8  
2026-03-26 19:29:00.047 +0330 [40142] LOG:  database system is ready to accept connections
```
«ریست شدم!»، خیلی هم بدردبخور :)). ولی ماجرا ‍‍`Segmentation fault`عه که طبیعی هم هست. ما به همه‌ی فانکشن‌هایی که یه کاری می‌کردن گفتیم مقدار صفر تایپ رو برگردون که با توجه به خطا یا پوینتر ناله و یا آدرس تخیلی داره. الان مشکل رو می‌دونیم چیه ولی یه مشکل ریز داریم، نمی‌دونیم مشکل کجاست. برای پیش رفتن ما دو راه داریم. راه اول اینه که `gdb` رو روشن کنیم و ببینیم کی داره می‌ترکه. باحاله ولی سخته. باید یه ریموت‌سرور بیاریم بالا و اینور ریکوئست بزنیم و اونور بترکیم.

 و راه ساده‌تر و پایتونی ماجرا. ما از فایل لاگ بالا یچی می‌فهمیم. سیستم تا قبل از اینکه بترکه لاگ میندازه. پس یه کاری که می‌تونیم بکنیم اینه که بریم تو سورس پسگره ببینیم چطور لاگ میندازه. وقتی فهمیدیم می‌تونیم داخل هر فانکشن یه لاگ بذاریم به اسم فانکشنی که کال شده و ببینیم تا کجا میره جلو. کثیفه ولی جواب میده. البته یه راه سوم هم هست که بریم دونه دونه کامنت‌های استراکت `TableAmRoutine` رو بخونیم و فانکشن‌ها رو بر اساس هش‌مپی که داریم پیاده‌سازی کنیم که صدالبته کی حوصله داره بره این همه راه رو.

برای پیدا کردن شیوه‌ی لاگ زدن، داخل سورس پسگره دنبال `checkpoint complete: wrote` از خطای بالا می‌گردیم و داخل فایل `src/backend/access/transam/xlog.c` حوالی خط ۶۸۳۹ میشه فانکشن `ereport` از سرآیند `elog` رو پیدا کرد. پس یه ماکرو تعریف می‌کنیم به اسم `flog`:
```diff
#include "fmgr.h"  
+#include "utils/elog.h"  
  
PG_MODULE_MAGIC;  
  
+#define flog elog(LOG, "*** function %s launched", __FUNCTION__)
```
و ازش داخل تمام فانکشن‌ها استفاده می‌کنیم (حاجی بد داره بمب می‌زنه الان :دی. ۶ فروردین، ۲۰:۰۲). برای مثال:
```diff
static const TupleTableSlotOps *  
hmam_slot_callbacks(Relation rel) {  
+   flog;  
    return NULL;  
}
```
## دومین خوندن از جدول
ام رو کامپایل و نصب می‌کنیم، فایل لاگ رو خالی می‌کنیم و دوباره کوئری سلکت رو ران می‌کنیم و میریم سروقت لاگ:
```
2026-03-26 20:18:33.619 +0330 [63548] LOG:  *** function hmam_tableam_handler launched at character 15  
2026-03-26 20:18:33.619 +0330 [63548] STATEMENT:  SELECT * FROM a;  
2026-03-26 20:18:33.619 +0330 [63548] LOG:  *** function hmam_relation_estimate_size launched  
2026-03-26 20:18:33.619 +0330 [63548] STATEMENT:  SELECT * FROM a;  
2026-03-26 20:18:33.622 +0330 [63548] LOG:  *** function hmam_slot_callbacks launched  
2026-03-26 20:18:33.622 +0330 [63548] STATEMENT:  SELECT * FROM a;  
2026-03-26 20:18:33.623 +0330 [40142] LOG:  client backend (PID 63548) was terminated by signal 11: Segmentation fault: 11  
2026-03-26 20:18:33.623 +0330 [40142] DETAIL:  Failed process was running: SELECT * FROM a;
```
عالی! یه جورایی مسیر خوندن رو هم انگار داریم می‌بینیم. به نظر می‌رسه که `hmam_slot_callbacks` کال شده و بعدش سرور کیل شده. اگه فایل هیپ و داکیومنت استراکت رو بخونیم، این فانکشن داره پیاده‌سازی‌های مرتبط با اسلات ام رو بر می‌گردونه.
### اسلات چیه؟
ولی سوال اینه که اسلات چیه؟ ولی قبل از اونکه به اسلات برسیم، بریم یه تصویر کلی از استوریج داخل پسگره از بالا بسازیم بریم پایین.
### ذخیره‌ی دیتا: از ماکرو تا میکرو
این نمودار می‌تونه یه مسیر روشن بهمون بده که ذخیره‌سازی داده از بالا به پایین اگه زوم کنیم چه شکلیه:
```
+-+-+-+-+-+-+-+-+-+
|   PG Instance   |
+-+-+-+-+-+-+-+-+-+
        /|\          <- has one or more
+-+-+-+-+-+-+-+-+-+
|    Database     |
+-+-+-+-+-+-+-+-+-+
        /|\
+-+-+-+-+-+-+-+-+-+
|     Schema      |
+-+-+-+-+-+-+-+-+-+
        /|\
+-+-+-+-+-+-+-+-+-+
|    Relation     |
+-+-+-+-+-+-+-+-+-+
        /|\
+-+-+-+-+-+-+-+-+-+
|      Fork       |
+-+-+-+-+-+-+-+-+-+
        /|\
+-+-+-+-+-+-+-+-+-+
|      Page       |
+-+-+-+-+-+-+-+-+-+
```
نقاشی‌مم بدم نیستا :)). وقتی یه نسخه‌ی پسگره نصب می‌کنیم (و یا با `pg_cluster‍` یه نسخه میاریم بالا، شبیه کاری که بالا با `initdb` و `pg_ctl‍` کردیم)، می‌تونیم داخلش یک یا چندتا دیتابیس داشته باشیم. هر دیتابیس هم طبیعتاً یک یا چندتا شِما داره، داخل هر شما هم یک یا چند ریلیشن وجود داره (که ریلیشن‌ها می‌تونن جدول، ایندکس، ویو و mv باشن)، هر ریلیشن هم روی یک یا چندتا فایل ذخیره میشه (هم از نظر تایپ هم از نظر تقسیم حجمی فایل) و در نهایت هر فایل به صورت بلاک‌های ۸ کیلوبایتی (به صورت دیفالت، تا ۳۲ کیلوبایت میشه افزایشش داد موقع کامپایل سرور پسگره) روی استوریج ذخیره میشن که بهشون میگیم پیج. این از یک نگاه بالا به پایین که با چی طرفیم.
### پیج
چیزی که الان مسئله‌ست اینه که پیج چیه. ساختار پیج این شکلیه:
```
+-+-+-+-+-+-+-+-+-+ <- 0
|     Header      |
+-+-+-+-+-+-+-+-+-+ <- 24
|  Item Pointers  |
+-+-+-+-+-+-+-+-+-+ <- lower
|    Free Space   |
+-+-+-+-+-+-+-+-+-+ <- upper
|      Items      |
+-+-+-+-+-+-+-+-+-+ <- special
|  Special Spaces |
+-+-+-+-+-+-+-+-+-+ <- pagesize (8192)
```
- ۲۴ بایت اول که هدره. هدر چک‌سام پیج و بقیه‌ی سایزها (که داخل نقاشی بالا با فلش مشخص شدن) رو ذخیره کرده. 
- از باید ۲۴ به بعد تا بایت `lower` یه آرایه از پوینترها ذخیره شده که به داده‌های واقعی (که بهشون میگیم تاپل و بر می‌گردیم بهش خیلی زود) اشاره می‌کنن. 
	- هر پوینتر ۴ بایته و شامل آفست تاپل از ابتدای پیج، طول تاپل و چند بیت که بیانگر وضعیت تاپل هست میشه. 
	- ضرورت حضور پوینترا برای اینه که ایندکس‌ها اگه بیان به آفست تاپل‌ها اشاره کنن، بعد از مثلاً وکیوم و جابجا شدن احتمالی اونا داخل پیج، باید بروز شن و این یعنی کار اضافه. و کار باحالی که پسگره کرده اینه که به جای اشاره‌ی مستقیم به خود تاپل، میاد به پوینتر اشاره می‌کنه و طبعاُ به صورت غیرمستقیم به خود تاپل با استفاده از پوینتر. اینطور اگه آفست تاپل هم داخل پیج عوض شه، ایندکس چیزی متوجه نمیشه و نیاز به تغییرش نیست.
- بعد از اون فضای خالیه که از بالا با اضافه شدن دیتا از طرف `lower` میاد پایین (مقدار `lower` به ازای هر پوینتر ۴ بایت بیشتر میشه) و از `upper` میاد بالا (مقدار `upper` کمتر میشه).
- از بایت `upper` تا `special` هم تاپل‌ها شروع میشن که تو بخش بعد دل و روده‌شون رو می‌ریزیم بیرون. 
- و در نهایت، از بایت `special` به بعد هم یه سری اطلاعات اضافی رو بعضی از ام‌ها ذخیره می‌کنن که روی هیپ صفره (می‌افته روی همون `pagesize‍`).

خود ساختار پیج هم خیلی کیوته. تایپش یه پوینتر به `char`عه که میشه ۸بیت. و بنابر سایزی که می‌خواد میاد از حافظه تخصیص می‌ده و مقدار صفر رو تو هر بایتش میذاره (یه جوری مثل استفاده از صورت پوینتری به جای کروشه داخل آرایه. تابع `PageInit` خط ۴۲ از `src/backend/storage/page/bufpage.c` یه جای خوبه برای شروع اینسپکت گرفتن در این مورد). بعد پوینتر رو تبدیل می‌کنه به ساختار پیج‌هدر و پیش میره.

جزئیات پیج رو میشه در فایل `src/include/storage/bufpage.h` دید و با اکستنشن `pageinspect` میشه آمار یه پیج رو در آورد (سورس این اکستنشن رو میشه در مسیر `contrib/pageinspect` دید). در ادامه ما یه جدول الکی با هیپ می‌سازیم و یچی توش می‌نویسیم و با این اکستنشن هم آمارش رو در میاریم.
```sql
postgres=# CREATE TABLE alaki (a integer);
CREATE TABLE
postgres=# INSERT INTO alaki (a) VALUES (42);
INSERT 0 1
postgres=# SELECT * FROM alaki;
 a
----
 42
(1 row)

postgres=# CREATE EXTENSION pageinspect;
CREATE EXTENSION
postgres=# SELECT lower, upper, special, pagesize FROM page_header(get_raw_page('alaki',0));
 lower | upper | special | pagesize
-------+-------+---------+----------
    28 |  8160 |    8192 |     8192
(1 row)

postgres=# INSERT INTO alaki (a) VALUES (16);
INSERT 0 1
postgres=# SELECT lower, upper, special, pagesize FROM page_header(get_raw_page('alaki',0));
 lower | upper | special | pagesize
-------+-------+---------+----------
    32 |  8128 |    8192 |     8192
(1 row)
```
تنها نکته اینکه وقتی از روی سورس پسگره رو کامپایل کنیم، اکستنشن‌ها کامپایل و نصب نمیشن و باید جدا اونا رو کامپایل و نصب کرد، یعنی `cd contrib && make all && sudo make install`.
### تاپل
و در نهایت می‌رسیم به تاپل که دیتای واقعی در اون ذخیره شده. تاپل از دو بخش تشکیل میشه، هدر و دیتا. ساختار هدر این شکلیه:
- فیلد t_heap: دوازده بایت.
	- فیلد t_xmin: چهار بایت. آیدی ترانزاشکن.
	- فیلد t_xmax: چهار بایت. آیدی ترانزاکشن.
	- فیلد t_cid: چهار بایت.
- فیلد t_ctid: شش بایت. همون پوینتر که بالا صبحت کردیم در موردش اینه. اینجا این پوینتر به تاپل بعدی که آپدیت این تاپله اشاره می‌کنه.
- فیلد t_infomask2: دو بایت. تعداد اتربیوت‌ها (۱۱ بیت پایین)، تاپل آپدیت یا حذف شده یا نه و موارد مرتبط با visibility رو نگه می‌داره.
- فیلد t_infomask: دو بایت. فلگ‌های اینکه در ستون‌هامون نال داره یا نه، toast شده یا نه، طول متغیر داریم یا نه و اینطور چیزا رو ذخیره می‌کنه.
- فیلد t_hoff: یک بایت. سایز هدر.
- فیلد t_bits: سایز متغیر.  بیت‌مپ مقادیر نال.
مشخصاً اگه مقدار نال نداشته باشیم، ۲۳ بایت طول هدر میشه. و خود تاپل هم ساختارش این شکلیه:
- فیلد t_len: چهار بایت. طول هدر رو مشخص می‌کنه.
- فیلد t_self: شش بایت. پوینتر به خودش (همون ساختار بالا و نه پوینتر سی).
- فیلد t_tableOid: چهار بایت. آیدی جدولی که این متعلق به اونه.
- فیلد t_data: که همون هدر بالاست.
بالاتر هم که دیدیم که، فضایی که برای یه پیج گرفته میشه در واقع یه آرایه از بایت‌هاست. یعنی دیتای خام در نهایت میاد در ادامه‌ی اون ۲۳ بایت هدر (اگه نال نداشته باشیم و اگه داشته باشیم بعد از اون بیت‌مپ) می‌شینه و با استفاده از فلگ‌ها پردازش میشه. در نهایت تاپل نمایش داده‌ایه که می‌خوایم ذخیره کنیم و از زرنگی پسگره، این نمایش برای هیپ داخل دیسک و مموری یکی هم هست.

تا اینجا که اومدیم، نه کامل (در یه پست دیگه بهش برگردیم)، جزئیات تاپل‌های هیپ رو میشه اینطور دید:
```sql
postgres=# CREATE TABLE alaki (a integer);
CREATE TABLE
postgres=# INSERT INTO alaki (a) VALUES (16), (32);
INSERT 0 2
postgres=# SELECT * FROM heap_page_items(get_raw_page('alaki',0));
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |   t_data
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+------------
  1 |   8160 |        1 |     28 |    816 |      0 |        0 | (0,1)  |           1 |       2048 |     24 |        |       | \x10000000
  2 |   8128 |        1 |     28 |    816 |      0 |        0 | (0,2)  |           1 |       2048 |     24 |        |       | \x20000000
(2 rows)

postgres=# BEGIN;
BEGIN
postgres=*# UPDATE alaki SET a = 10 WHERE a = 16;
UPDATE 1
postgres=*# SELECT * FROM heap_page_items(get_raw_page('alaki',0));
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |   t_data
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+------------
  1 |   8160 |        1 |     28 |    816 |    817 |        0 | (0,3)  |       16385 |        256 |     24 |        |       | \x10000000
  2 |   8128 |        1 |     28 |    816 |      0 |        0 | (0,2)  |           1 |       2304 |     24 |        |       | \x20000000
  3 |   8096 |        1 |     28 |    817 |      0 |        0 | (0,3)  |       32769 |      10240 |     24 |        |       | \x0a000000
(3 rows)

postgres=*# UPDATE alaki SET a = 16 WHERE a = 36;
UPDATE 0
postgres=*# UPDATE alaki SET a = 16 WHERE a = 32;
UPDATE 1
postgres=*# SELECT * FROM heap_page_items(get_raw_page('alaki',0));
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |   t_data
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+------------
  1 |   8160 |        1 |     28 |    816 |    817 |        0 | (0,3)  |       16385 |        256 |     24 |        |       | \x10000000
  2 |   8128 |        1 |     28 |    816 |    817 |        2 | (0,4)  |       16385 |        256 |     24 |        |       | \x20000000
  3 |   8096 |        1 |     28 |    817 |      0 |        0 | (0,3)  |       32769 |      10240 |     24 |        |       | \x0a000000
  4 |   8064 |        1 |     28 |    817 |      0 |        2 | (0,4)  |       32769 |      10240 |     24 |        |       | \x10000000
(4 rows)

postgres=*# COMMIT;
COMMIT
postgres=# SELECT * FROM heap_page_items(get_raw_page('alaki',0));
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |   t_data
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+------------
  1 |   8160 |        1 |     28 |    816 |    817 |        0 | (0,3)  |       16385 |        256 |     24 |        |       | \x10000000
  2 |   8128 |        1 |     28 |    816 |    817 |        2 | (0,4)  |       16385 |        256 |     24 |        |       | \x20000000
  3 |   8096 |        1 |     28 |    817 |      0 |        0 | (0,3)  |       32769 |      10240 |     24 |        |       | \x0a000000
  4 |   8064 |        1 |     28 |    817 |      0 |        2 | (0,4)  |       32769 |      10240 |     24 |        |       | \x10000000
(4 rows)
```

جزئیات تاپل رو میشه در `src/include/executor/tuptable.h` و `src/include/access/htup.h` و `src/include/access/htup_details.h` دید.
### خب اسلات چیه؟
اسلات خودش یه رپر دور تاپله که یه سری اطلاعات در مورد تاپل (مثل تعداد اتریبیوت‌ها و اینطور چیزا) رو نگه می‌داره و به همراه یه سری فانکشن که کارای مدیریت منابع رو برای تاپل‌ها هندل می‌کنه جنرالی میشه پلی بین تاپل به عنوان دیتای خام و لایه‌ی بالا به عنوان اجرای کوئری و اینطور چیزا. در حال حاضر پسگره چهار تایپ اسلات رو ساپورت می‌کنه:
- تایپ `TTSOpsBufferHeapTuple`: این تایپ مدیریت بافری که از دیسک خونده شده رو به عهده می‌گیره.
- تایپ `TTSOpsHeapTuple`: این تایپ روی مموری کارا رو هندل می‌کنه و مثل قبلی حواسش به پین بودن پیج تا تموم شد کار نیست.
- تایپ `TTSOpsMinimalTuple`: اینو نفهمیدم و الان برام مهم نبود برم دنبالش.
- تایپ `TTSOpsVirtual`: این هم برای لایه‌های میانی اجراست. مثلاً یه اسلات پایین‌دستی داره خوندن تاپل از دیسک رو هندل می‌کنه. برای اینکه تاپل رو به صورت فیزیک کپی نکنیم هی، این میاد یه شکلی پوینتز میشه به پایینی.

برای پیش رفتن کارمون پس ما `TTSOpsHeapTuple` رو بر میداریم و به فانکشنی که خطا خورده بود میدیم:
```diff
static const TupleTableSlotOps *  
hmam_slot_callbacks(Relation rel) {  
    flog;  
+   return &TTSOpsHeapTuple;  
}
```
و میریم برای ران کردن دوباره‌ی کوئری‌مون.
## سومین خوندن از جدول
طبعاً کوئری خطا داده و سرور ریست شده. لاگ اینه:
```
2026-03-28 21:09:05.595 +0330 [53019] LOG:  *** function hmam_tableam_handler launched at character 15  
2026-03-28 21:09:05.595 +0330 [53019] STATEMENT:  SELECT * FROM a;  
2026-03-28 21:09:05.595 +0330 [53019] LOG:  *** function hmam_relation_estimate_size launched  
2026-03-28 21:09:05.595 +0330 [53019] STATEMENT:  SELECT * FROM a;  
2026-03-28 21:09:05.596 +0330 [53019] LOG:  *** function hmam_slot_callbacks launched  
2026-03-28 21:09:05.596 +0330 [53019] STATEMENT:  SELECT * FROM a;  
2026-03-28 21:09:05.596 +0330 [53019] LOG:  *** function hmam_scan_begin launched  
2026-03-28 21:09:05.596 +0330 [53019] STATEMENT:  SELECT * FROM a;
```
این بار تابع `hmam_scan_begin` سالم در نیومده. تابع باید یه استراکت `TableScanDesc` برگردونه و که این استراکت ریلیشن رو مشخص می‌کنه و ستون‌هایی که احتمالاً نیازه روشون فیلتر انجام بشه و اسنپ‌شاتی که اسکن روی اون شکل می‌گیره. ما هدفمون اینه امی که می‌خوایم پیاده‌سازی کنیم فقط اسکن خطی رو پشتیبانی کنه. برای شروع سرآیند موردنیاز رو اضافه می‌کنیم:
```diff 
#include "utils/elog.h"  
+#include "access/heapam.h"  
  
PG_MODULE_MAGIC;
```
و میریم برای کپی کردن متد هیپ و تغییرش برای استوریج خودمون :)).
```c
static TableScanDesc  
hmam_scan_begin(Relation rel,  
                Snapshot snapshot,  
                int nkeys, ScanKeyData *key,  
                ParallelTableScanDesc pscan,  
                uint32 flags) {  
    flog;  
    HeapScanDesc scan = (HeapScanDesc) palloc_object(HeapScanDescData);  
  
    scan->rs_base.rs_rd = rel;  
    scan->rs_base.rs_snapshot = snapshot;  
    scan->rs_base.rs_nkeys = nkeys;  
    scan->rs_base.rs_flags = flags;  
    scan->rs_base.rs_parallel = NULL;  
    scan->rs_strategy = NULL;  
    scan->rs_cbuf = InvalidBuffer;  
  
    if (scan->rs_base.rs_flags & (SO_TYPE_SEQSCAN | SO_TYPE_SAMPLESCAN)) {  
        Assert(snapshot);  
        PredicateLockRelation(rel, snapshot);  
    }  
  
    scan->rs_ctup.t_tableOid = RelationGetRelid(rel);  
    scan->rs_parallelworkerdata = NULL;  
  
    if (nkeys > 0)  
        scan->rs_base.rs_key = palloc_array(ScanKeyData, nkeys);  
    else  
        scan->rs_base.rs_key = NULL;  
  
    scan->rs_read_stream = NULL;  
  
    return (TableScanDesc) scan;  
}
```
کامپایل و نصب می‌کنیم و میریم برای ران بعدی.
## چهارمین خوندن از جدول
کوئری‌مون رو دوباره اجرا می‌کنیم:
```sql
postgres=# SELECT * FROM a;
 b
---
(0 rows)
```
و خبر خوب! ران شد :)). لاگ اجرامون هم اینه:
```
2026-03-28 21:41:35.895 +0330 [60326] LOG:  *** function hmam_tableam_handler launched at character 15  
2026-03-28 21:41:35.895 +0330 [60326] STATEMENT:  SELECT * FROM a;  
2026-03-28 21:41:35.896 +0330 [60326] LOG:  *** function hmam_relation_estimate_size launched  
2026-03-28 21:41:35.896 +0330 [60326] STATEMENT:  SELECT * FROM a;  
2026-03-28 21:41:35.899 +0330 [60326] LOG:  *** function hmam_slot_callbacks launched  
2026-03-28 21:41:35.899 +0330 [60326] STATEMENT:  SELECT * FROM a;  
2026-03-28 21:41:35.899 +0330 [60326] LOG:  *** function hmam_scan_begin launched  
2026-03-28 21:41:35.899 +0330 [60326] STATEMENT:  SELECT * FROM a;  
2026-03-28 21:41:35.899 +0330 [60326] LOG:  *** function hmam_scan_getnextslot launched  
2026-03-28 21:41:35.899 +0330 [60326] STATEMENT:  SELECT * FROM a;  
2026-03-28 21:41:35.899 +0330 [60326] LOG:  *** function hmam_scan_end launched  
2026-03-28 21:41:35.899 +0330 [60326] STATEMENT:  SELECT * FROM a;
```
الان دوتا مسیر رو می‌تونیم بریم. اول اینکه ادامه‌ی خط تابع‌هایی که کال شدن رو بگیریم و کامل کنیم ولی مشکلی که هست اینه که دیتایی نداریم که هر بار بتونیم یه سنجه‌ای داشته باشیم برای درست بودن کارمون. پس اول می‌ریم سر وقت وارد کردن دیتا داخل جدول.
## اولین ورود دیتا
یه دیتای ساده وارد می‌کنیم:
```sql
postgres=# INSERT INTO a (b) VALUES (16);
INSERT 0 1
```
و لاگ‌مون:
```
2026-03-28 21:58:00.622 +0330 [61530] LOG:  *** function hmam_slot_callbacks launched  
2026-03-28 21:58:00.622 +0330 [61530] STATEMENT:  INSERT INTO a (b) VALUES (16);  
2026-03-28 21:58:00.622 +0330 [61530] LOG:  *** function hmam_tuple_insert launched  
2026-03-28 21:58:00.622 +0330 [61530] STATEMENT:  INSERT INTO a (b) VALUES (16);
```
چه جالب! خطا نگرفتیم. میریم برای سلکت:
```sql
postgres=# SELECT * FROM a;
 b
---
(0 rows)
```
عالی. به نظر کاری که باید بکنیم پیاده‌سازی `hmam_tuple_insert`عه. 
### پایان یک رویا
یه مشکلی که الان می‌تونیم متوجهش شیم اینه که ما به صورت خیلی معصومانه‌ای، یه هش‌مپ ساختیم و خواستیم کل دیتا رو داخل اون ذخیره کنیم که یه اینتیجر بود. ولی الان می‌دونیم که کمینه حالتی که می‌تونیم داده رو ذخیره کنیم یه شکلی از تاپله (که می‌تونیم خودمون پیاده کنیم یا از تاپلی که داریم استفاده کنیم که فعلاً دومی رو انتخاب می‌کنیم). جدا از اون هش‌مپ یه مشکل دیگه هم داره. اول کار اومدیم یه هش‌مپ ساختیم که بر مبنای یه کلید، داخل یه لیست ذخیره میشه. هیچ خبری از تفاوت جدول‌ها نیست.
### هش‌مپی برای تموم فصول
پس به نظر می‌رسه که باید یه کم ماجرا رو تغییر بدیم. اول اینکه یه هش‌مپ داشته باشیم که کلیدش آیدی جدوله و مقدارش یه هش‌مپ دیگه که ولیوهای اون جدول داخلش ذخیره میشن. و دوم اینکه ولیوهای جدول دوم تایپ‌شون باید تاپل باشه و نه اینتیجر. لیست پیوندی‌مون که تقریباً تغییری نمی‌کنه. سرآیندش:
```diff
 struct LinkedListData {
     int key;
-    int value;
+    void *value;
     LinkedList next;
 };

-LinkedList create_linkedlist_node(int key, int value);
+LinkedList create_linkedlist_node(int key, void *value);

-LinkedList insert_linkedlist(LinkedList ll, int key, int value);
+LinkedList insert_linkedlist(LinkedList ll, int key, void *value);

 LinkedList find_linkedlist(LinkedList ll, int key);
```
و کدش:
```diff
 #include <stdlib.h>
 #include "linkedlist.h"

-LinkedList create_linkedlist_node(const int key, const int value) {
+LinkedList create_linkedlist_node(const int key, void *value) {
     LinkedList ll = malloc(sizeof(struct LinkedListData));
     ll->key = key;
     ll->value = value;


-LinkedList insert_linkedlist(LinkedList ll, const int key, const int value) {
+LinkedList insert_linkedlist(LinkedList ll, const int key, void *value) {
     LinkedList current;
     for (current = ll; current != NULL; current = current->next) {
         if (current->key == key) {


bool delete_linkedlist(LinkedList ll, const int key) {  
    for (LinkedList current = ll, prev = ll; current != NULL; prev = current, current = current->next) {  
        if (current->key == key) {  
            prev->next = current->next;  
+           free(current->value);  
            free(current); 
}
```
فقط تایپ المنت رو به پوینتر `void` تغییر دادیم. پوینتر `void` مثل اینترفیس خالی داخل گو یا تایپ‌اسکریپت می‌مونه و می‌تونه هر پوینتر دیگه‌ای رو بپذیره. موقع حذف هم حواسمون هست که مقدار `value` چون پوینتر شده، اون رو آزاد کنیم. تغییرات سرآیند هش‌مپ:
```diff
 #include <stdbool.h>
+#include "linkedlist.h"

-#define MAX_SIZE 100
+typedef struct HashmapData *Hashmap;

-LinkedList hashmap[MAX_SIZE];
+struct HashmapData {
+    int bs; // bucket size
+    LinkedList *ll;
+};

-int hash(int key);
+Hashmap create_hashmap(int bucket_size);

-void insert_hashmap(int key, int value);
+int hash(Hashmap hm, int key);

-bool lookup_hashmap(int key, int *value);
+void insert_hashmap(Hashmap hm, int key, void *value);

-int delete_hashmap(int key);
+bool lookup_hashmap(Hashmap hm, int key, void **value);

- bool contain_hashmap(int key);
+ bool contain_hashmap(Hashmap hm, int key);

+
+int delete_hashmap(Hashmap hm, int key);
```
که هش‌مپ اینجا دیگه یه آرایه با یه باکت مشخص نیست و استراکت میشه که برای خودش باکت‌سایز داره و لیست پیوندیش هم یه آرایه به پوینترهای لیست‌پیوندیه. و کد هش‌مپ:
```diff
+#include <stdlib.h>
 #include <stddef.h>
 #include "linkedlist.h"
 #include "hashmap.h"

-int hash(const int key) {
-    return key % MAX_SIZE;
+Hashmap create_hashmap(int bucket_size) {
+    Hashmap hm = malloc(sizeof(struct HashmapData));
+    hm->bs = bucket_size;
+    hm->ll = malloc(sizeof(LinkedList) * bucket_size);
+
+    return hm;
+}
+
+int hash(Hashmap hm, const int key) {
+    return key % hm->bs;
 }

-void insert_hashmap(int key, int value) {
-    LinkedList ll = hashmap[hash(key)];
+void insert_hashmap(Hashmap hm, int key, void *value) {
+    LinkedList ll = hm->ll[hash(hm, key)];
     if (ll == NULL) {
         LinkedList node = create_linkedlist_node(key, value);
-        hashmap[hash(key)] = node;
+        hm->ll[hash(hm, key)] = node;
         return;
     }

-    insert_linkedlist(hashmap[hash(key)], key, value);
+    insert_linkedlist(hm->ll[hash(hm, key)], key, value);
 }

-bool lookup_hashmap(int key, int *value) {
-    LinkedList ll = find_linkedlist(hashmap[hash(key)], key);
+bool lookup_hashmap(Hashmap hm, int key, void **value) {
+    LinkedList ll = find_linkedlist(hm->ll[hash(hm, key)], key);
     if (ll != NULL) {
         *value = ll->value;
         return true;
     }
     return false;
 }
 
-bool contain_hashmap(int key) {
-	LinkedList ll = find_linkedlist(hashmap[hash(key)], key);  
+bool contain_hashmap(Hashmap hm, int key) {  
+    if (hm == NULL || hm->ll == NULL || hm->ll[hash(hm, key)] == NULL) {  
+        return false;  
+    }  
  
+    LinkedList ll = find_linkedlist(hm->ll[hash(hm, key)], key);  
    return ll != NULL;  
}

-int delete_hashmap(int key) {
-    return delete_linkedlist(hashmap[hash(key)], key);
+int delete_hashmap(Hashmap hm, int key) {
+    return delete_linkedlist(hm->ll[hash(hm, key)], key);
 }
```
تنها نکته‌ای که این وسط شاید اگه سی کار نکرده باشیم یه کم عجیب باشه داخل فانکشن `lookup_hashmap` هست. `void **value` یه پوینتره به پوینتری که `void`عه. گفتیم که `void *value` هر پوینتری رو می‌پذیره. حالا ما می‌خوایم اگه سرچ‌مون نتیجه داشت، مقداری که از لیست‌پیوندی‌مون (که خودش یه پوینتره) رو بذاریم داخل `value`. ولی `void *value` رو داریم که پوینتره و دی‌رفرنس کردنش میشه خود مقدار و مقدار پوینتر رو نمی‌شه بهش اساین کرد. 
### تغییر آدرس پوینتر
قبل از اینکه بریم سر وقت روشن شدن ماجرا، ما می‌دونیم که پوینتر، خونه‌ای از حافظه‌ست که داخلش آدرس یه جای دیگه از حافظه ذخیره میشه. پس محتویاتش یه آدرسه و چون خودش هم یه خونه از حافظه‌ست، پس یه آدرس داره. 

حالا برای روشن شدن ماجرا این کد رو در نظر بگیریم:
```c
#include <stdio.h>  
  
struct MyData {  
    int a;  
};  
  
typedef struct MyData MyData;  
  
void print(MyData *m) {  
    printf("a = %d\n", m->a);  
}  
  
void increment_by_one(MyData *v) {  
    MyData my = {.a = v->a + 1};  
    v = &my;  
    print(v);  
}  
  
int main(void) {  
    MyData b = {.a = 42};  
    print(&b);  
    increment_by_one(&b);  
    print(&b);  
  
    return 0;  
}
```
و کد رو ران می‌کنیم:
```shell
meysam@Meysams-Mac pgam % gcc -Wall -o test2 test2.c
meysam@Meysams-Mac pgam % ./test2
a = 42
a = 43
a = 42
```
مقدار `a` داخل فانکشن `increment_by_one` عوض شد ولی بیرون همون مقدار قبلی موند. اگه اسمبلی فانکشن `increment_by_one` رو ببینیم دلیل روشن میشه:
```asm {linenos=inline hl_lines=[5,"8-11"]}
_increment_by_one:                      
	sub	sp, sp, #32
	stp	x29, x30, [sp, #16]             
	add	x29, sp, #16
	str	x0, [sp, #8] <- store x0 (pointer to the struct) in sp+8
	ldr	x8, [sp, #8]
	ldr	w8, [x8]
	add	w9, w8, #1   <- do a++
	add	x8, sp, #4   <- move x8 address to sp+4
	str	w9, [sp, #4] <- store result on sp+4
	str	x8, [sp, #8] <- store result (x0 points to sp+4) in sp+8
	ldr	x0, [sp, #8]
	bl	_print
	ldp	x29, x30, [sp, #16]             
	add	sp, sp, #32
	ret
```
خط ۵ تا ۱۱ ماجرا رو روشن می‌کنه. تو پست [محاسبه‌گر نت‌مسک در اسمبلی آرم۶۴](blog/write-a-netmask-calculator-in-arm64-asm/) دیدیم که طی قرارداد، رجیستر `x0` به عنوان آرگومان اول تابع استفاده می‌شه (در اینجا استراکچری که پاس دادیم). در خط ۵ میاد اون رو داخل استک ذخیره می‌کنه (پس ما الان آدرس استراکت رو داخل استک داریم)، خط ۶ میاد آدرس رو از داخل استک می‌خونه و داخل رجیستر `x8` می‌ریزه و خط ۷ مقدار ذخیره شده در اون آدرس رو می‌خونه و داخل ورد `w8` می‌ریزه. اینجا همون جاییه که ما ارتباط‌مون رو با آدرس اصلی‌ای که استراکت‌مون رو ذخیره کرد بود از دست میدیم و فقط داخل استک اون آدرس رو داریم. بعد از انجام محاسبات هم نتیجه رو در خط ۱۱ داخل استک جایی که قبلاُ آدرس استراکت‌مون ذخیره می‌کنه. پس همونطور که دیدیم داخل استراکتی که پاس دادیم هیچ اتفاقی نیفتاد عملاً.

حالا اگه بیایم و به جای پاس دادن آدرس استراکت، آدرسِ آدرسِ استراکت رو پاس بدیم چی میشه؟ (یادمون هست که چون پوینتر یه خونه از حافظه‌ست پس خودش هم یه آدرسی داره).
```c
#include <stdio.h>  
  
struct MyData {  
    int a;  
};  
  
typedef struct MyData MyData;  
  
void print(MyData *m) {  
    printf("a = %d\n", m->a);  
}  
  
void increment_by_one(MyData **v) {  
    MyData my = {.a = (*v)->a + 1};  
    *v = &my;  
    print(*v);  
}  
  
int main(void) {  
    MyData b = {.a = 42};  
    MyData *c = &b;  
    print(c);  
    increment_by_one(&c);  
    print(c);  
  
    return 0;  
}
```
و اجراش:
```shell
meysam@Meysams-Mac pgam % gcc -Wall -o test2 test2.c
meysam@Meysams-Mac pgam % ./test2
a = 42
a = 43
a = 43
```
درست کار کرد! بریم کد اسمبلی‌ش رو ببینیم:
```asm {linenos=inline hl_lines=["11-13"]}
_increment_by_one:                      
	sub	sp, sp, #32
	stp	x29, x30, [sp, #16]             
	add	x29, sp, #16
	str	x0, [sp, #8] <- store the address of struct's address in sp+8
	ldr	x8, [sp, #8] <- load the address of the struct
	ldr	x8, [x8]     <- load the struct
	ldr	w8, [x8]     <- load a from struct
	add	w9, w8, #1   <- a++
	add	x8, sp, #4   <- set x8 to address of sp+4
	str	w9, [sp, #4] <- store a in sp+4
	ldr	x9, [sp, #8] <- load the address of struct's address from sp+8
	str	x8, [x9]     <- store x8 (is an address) into x9
	ldr	x8, [sp, #8]
	ldr	x0, [x8]
	bl	_print
	ldp	x29, x30, [sp, #16]             
	add	sp, sp, #32
	ret
```
اینجا طبعاً اول کار که کدها شبیه اسمبلی قبلیه ولی تفاوت در خط ۱۱ تا ۱۳ مشخص میشه. آدرس استراکت جدید داخل استک+۴ ذخیره میشه. مقدار آدرسِ آدرسِ استراکت که داخل استک+۸ ذخیره شده فراخوانی میشه و داخل اون آدرس، آدرس استراکت جدید ذخیره میشه :)). پس برای جمع‌بندی، پوینتر اینطور چیزیه:
```
+-------+         +-------+ 
|   B   | ------> |   C   |
+-------+         +-------+
```
ما وقتی `B` رو داشته باشیم، می‌تونیم محتویات جایی که بهش اشاره می‌کنه یعنی `C` رو ببینیم یا تغییر بدیم. و پوینتر به پوینتر اینطور چیزی:
```
+-------+         +-------+         +-------+ 
|   A   | ------> |   B   | ------> |   C   |
+-------+         +-------+         +-------+
```
و طبعاً وقتی ما `A` رو داشته باشیم، می‌تونیم محتویات جایی که بهش اشاره می‌کنه، یعنی `B` رو ببینیم و تغییرش بدیم. محتویات `B` هم که آدرس به یه خونه‌ی حافظه‌ست، پس می‌تونیم بگیم به جای دیگه اشاره کنه:
```
+-------+         +-------+         +-------+ 
|   A   | ------> |   B   |         |   C   |
+-------+         +-------+         +-------+
                      |
                      |             +-------+
                      +-----------> |   D   |
                                    +-------+
```
این کل کاریه که اینجا اتفاق افتاد.
### ابسترکت کردن نوشتن روی هش‌مپ
دیدیم که یه سری مشکلات مثل گرفتن جدول از دیتابیس داخل هش‌مپ‌مون وجود داره و هر بار باید اون کار رو انجام بدیم. اینه که بد نیست یه ابسترکشن برای دیتابیس و جدول هم داشته باشیم و کل استفاده داخل ام رو محدود کنیم به این ابسترکشن. این سرآیند:
```c
#pragma once  
#include "postgres.h"  
#include "hashmap.h"  
#include "access/heapam.h"  
  
static Hashmap hmdb;  
  
void init_hm_db(void);  
  
void init_hm_table(Relation rel);  
  
void insert_hm_dbvoid insert_hm_db(Relation rel, HeapTuple tup);  
  
Hashmap get_hm_db(Relation rel);
```
و این هم پیاده‌سازیش:
```c
#include <stdlib.h>  
  
#include "postgres.h"  
#include "hashmap.h"  
#include "hmops.h"  
#include "access/tableam.h"
  
void init_hm_db(void) {  
    if (hmdb != NULL)  
        return;  
  
    hmdb = create_hashmap(10);  
}  
  
void init_hm_table(Relation rel) {  
    unsigned int oid = rel->rd_id;  
    Hashmap hm = create_hashmap(100);  
    if (!lookup_hashmap(hmdb, oid, (void **)&hm)) {  
        insert_hashmap(hmdb, oid, hm);  
        return;  
    }  
    free(hm);  
}  
  
Hashmap get_hm_db(Relation rel) {  
    Hashmap hm;  
    if (lookup_hashmap(hmdb, rel->rd_id, (void **)&hm)) {  
        return hm;  
    }  
  
    return NULL;  
}  
  
void insert_hm_db(Relation rel, HeapTuple tup) {  
    Hashmap hm = get_hm_db(rel);  
    insert_hashmap(hm, 42, tup);  
}
```
فعلاً کلید رو گذاشتم ۴۲ که بتونیم بریم جلو. بر می‌گردیم و درستش می‌کنیم.

خب در نهایت اگه بریم داخل کد ام‌مون می‌تونیم بریم برا پیاده‌سازی اینزرت. وارد کرد سرآیند:
```diff
#include "storage/predicate.h"
+#include "hmops.h" 
  
PG_MODULE_MAGIC;
```
بالا آوردن هش‌مپ اولیه:
```diff
Datum  
hmam_tableam_handler(PG_FUNCTION_ARGS) {  
    flog;  
+   init_hm_db(); 
    PG_RETURN_POINTER(&hmam_methods);  
}
```
مطمئن شدن از اینکه جدولی که می‌خوایم چیزی بهش وارد کنیم، هش‌مپ خودش رو داره:
```diff
static const TupleTableSlotOps *  
hmam_slot_callbacks(Relation rel) {  
    flog;  
+   init_hm_table(rel);  
    return &TTSOpsHeapTuple;  
}
```
و پیاده‌سازی `hmam_tuple_insert`:
```c
static void  
hmam_tuple_insert(Relation rel, TupleTableSlot *slot,  
                  CommandId cid, int options,  
                  BulkInsertStateData *bistate) {  
    flog;  
    bool shouldFree = true;  
    HeapTuple tuple = ExecFetchSlotHeapTuple(slot, true, &shouldFree);  
  
    /* Update the tuple with table oid */  
    slot->tts_tableOid = RelationGetRelid(rel);  
    tuple->t_tableOid = slot->tts_tableOid;  
  
    /* Perform the insertion, and copy the resulting ItemPointer */  
    insert_hm_db(rel, tuple, cid, options, bistate);  
    ItemPointerCopy(&tuple->t_self, &slot->tts_tid);  
  
    if (shouldFree)  
        pfree(tuple);  
}
```
بریم برای کامپایل و نصب:
```shell
meysam@Meysams-Mac pgam % make clean && make && sudo make install
rm -f hmam.dylib hmam.o  \
	    hmam.bc
rm -f hashmap.o linkedlist.o hashmap.bc linkedlist.bc
gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Werror=unguarded-availability-new -Wendif-labels -Wmissing-format-attribute -Wcast-function-type -Wformat-security -Wmissing-variable-declarations -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-unused-command-line-argument -Wno-compound-token-split-by-macro -Wno-format-truncation -Wno-cast-function-type-strict -g -O0 -g3 -fno-omit-frame-pointer  -fvisibility=hidden -I. -I./ -I/usr/local/pgsql/include/server -I/usr/local/pgsql/include/internal -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX26.2.sdk   -I/opt/homebrew/Cellar/icu4c@78/78.1/include    -c -o hmam.o hmam.c
hmam.c:28:18: warning: mixing declarations and code is incompatible with standards before C99 [-Wdeclaration-after-statement]
   28 |     HeapScanDesc scan = (HeapScanDesc) palloc_object(HeapScanDescData);
      |                  ^
hmam.c:122:10: warning: mixing declarations and code is incompatible with standards before C99 [-Wdeclaration-after-statement]
  122 |     bool shouldFree = true;
      |          ^
2 warnings generated.
gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Werror=unguarded-availability-new -Wendif-labels -Wmissing-format-attribute -Wcast-function-type -Wformat-security -Wmissing-variable-declarations -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-unused-command-line-argument -Wno-compound-token-split-by-macro -Wno-format-truncation -Wno-cast-function-type-strict -g -O0 -g3 -fno-omit-frame-pointer  -fvisibility=hidden hmam.o -L/usr/local/pgsql/lib -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX26.2.sdk   -Wl,-dead_strip_dylibs   -fvisibility=hidden -bundle -bundle_loader /usr/local/pgsql/bin/postgres -o hmam.dylib
Undefined symbols for architecture arm64:
  "_init_hm_db", referenced from:
      _hmam_tableam_handler in hmam.o
  "_init_hm_table", referenced from:
      _hmam_slot_callbacks in hmam.o
  "_insert_hm_db", referenced from:
      _hmam_tuple_insert in hmam.o
ld: symbol(s) not found for architecture arm64
clang: error: linker command failed with exit code 1 (use -v to see invocation)
make: *** [hmam.dylib] Error 1
```
چه عالی. خطا داریم. و خطا از سر اینه که نتونسته سرآیندا رو تشخیص بده. اگه به `Makefile`ی که اول کار شروع به ساختن ازش کردیم برگردیم میگه اونجا که چندتا فایله باید از `MOUDLE_big` استفاده کنیم و نه `MOUDLES`. پس بریم برای تغییر `Makefile`مون:
```make
MODULE_big = hmam  
OBJS = \  
    linkedlist.o \  
    hashmap.o \  
    hmops.o \  
    hmam.o  
  
EXTENSION = hmam  
  
PG_CONFIG = /usr/local/pgsql/bin/pg_config  
PGXS := $(shell $(PG_CONFIG) --pgxs)  
include $(PGXS)
```
و این بار اگه بزنیم کامپایل و نصب شه، همه چی اوکیه. و تستش:
```sql
postgres=# INSERT INTO a (b) VALUES (16);
INSERT 0 1
postgres=# SELECT * FROM a;
 b
---
(0 rows)
```
خطا نداد ولی چیزی هم نشون نداد! :)).
## اولین نمایش
ما تونستیم بدون خطا دیتا وارد کنیم، ولی روی کوئری‌ای که زدیم چیزی نگرفتیم. بر می‌گردیم به لاگ:
```
2026-03-30 14:59:21.135 +0330 [60309] LOG:  *** function hmam_tableam_handler launched at character 13  
2026-03-30 14:59:21.135 +0330 [60309] STATEMENT:  INSERT INTO a (b) VALUES (16);  
2026-03-30 14:59:21.140 +0330 [60309] LOG:  *** function hmam_slot_callbacks launched  
2026-03-30 14:59:21.140 +0330 [60309] STATEMENT:  INSERT INTO a (b) VALUES (16);  
2026-03-30 14:59:21.140 +0330 [60309] LOG:  *** function hmam_tuple_insert launched  
2026-03-30 14:59:21.140 +0330 [60309] STATEMENT:  INSERT INTO a (b) VALUES (16);  
2026-03-30 14:59:24.778 +0330 [60309] LOG:  *** function hmam_relation_estimate_size launched  
2026-03-30 14:59:24.778 +0330 [60309] STATEMENT:  SELECT * FROM a;  
2026-03-30 14:59:24.784 +0330 [60309] LOG:  *** function hmam_slot_callbacks launched
2026-03-30 14:59:24.784 +0330 [60309] STATEMENT:  SELECT * FROM a;  
2026-03-30 14:59:24.784 +0330 [60309] LOG:  *** function hmam_scan_begin launched  
2026-03-30 14:59:24.784 +0330 [60309] STATEMENT:  SELECT * FROM a;  
2026-03-30 14:59:24.784 +0330 [60309] LOG:  *** function hmam_scan_getnextslot launched  
2026-03-30 14:59:24.784 +0330 [60309] STATEMENT:  SELECT * FROM a;  
2026-03-30 14:59:24.784 +0330 [60309] LOG:  *** function hmam_scan_end launched  
2026-03-30 14:59:24.784 +0330 [60309] STATEMENT:  SELECT * FROM a;
```
بعد از اینکه اسکن شروع میشه تابع `hmam_scan_getnextslot` کال میشه. به نظر میاد که اون چیزیه که باید بریم سر وقتش. ولی به نظرم قبلش برای اینکه بفهمیم پارامترهای این تابع چی‌ان و چطور باید ازشون استفاده کنیم، بد نیست که بفهمیم این تابع قراره از کجا کال شه. برای این منظور می‌ریم ببینیم وقتی کوئری سلکت‌مون رو ران می‌کنیم داخل پسگره چه اتفاقی می‌افته و چه مسیری رو می‌ره.
## مسیر یک اجرا
فرض کنیم ما می‌زنیم سرور ران شه و یه سری پروسه ران میشن و سرور بالا میاد و ماجرا استیبل میشه. می‌خوایم ببینیم وقتی به عنوان کلاینت به سرور ریکوئست می‌زنیم چه اتفاقی می‌افته.  برای خوندن این بخش نیاز به سورس پسگره روی هشی که اول کار نوشتم داریم. و چون برای خودم جذابه اینکه ببینم توی چیزا چه خبره، احتمالاً این بخش یه کم بلنده میشه و کد خاصی هم نداره ولی دید خوبی میده از خود پسگره.
### لوپ‌های بی‌نهایت
بعد از اینکه سرور پسگره بالا اومد، تابع `ServerLoop` در خط ۱۶۶۳ فایل `src/backend/postmaster/postmaster.c`  ران میشه. همونطور که از اسمش بر میاد این تابع یه لوپ بی‌نهایته. حالا با فرض اینکه مشکلی نیست و ظرفیت رم و اینا اوکی‌ان:
1. کلاینت ما یه کانکشن به سرور می‌زنه و کانکشن برقرار میشه.
2. در تکرار بعدی سرورلوپ، برای ریکوئست ما تابع `BackendStartup` در خط ۱۷۱۳ ران میشه.
3. اگه مشکلی نباشه تابع `BackendMain` در فایل `src/backend/tcop/backend_startup.c` به عنوان یه پروسه‌ی جدید فورک میشه و ران میشه.
4. از اونجا تابع `PostgresMain` از فایل `src/backend/tcop/postgres.c` ران میشه. داخل این تابع در ابتدا یه سری سیگنال هندلر و تایم‌اوت ست میشن.
5. داخل ۴ خط ۴۲۹۸ تابع `InitPostgres`(از فایل `src/backend/utils/init/postinit.c` خط ۷۱۰) کال میشه و سشن مسیر دیتابیس و اینطور چیزا رو ست می‌کنه و چک می‌کنه که دیتابیس وجود داشته باشه و همه چی اگه اوکی باشه سشن رو آغاز می‌کنه و بر می‌گرده.
6.  در ادامه‌ی تابع `PostgresMain` یوزر اگه سوپریوزر نباشه در ادامه‌ی  می‌ره برای احراز هویت.
7. تا اینجا اگه همه چی اوکی باشه، داخل فانکشن `PostgresMain` 
	1. خط ۴۴۰۲ میاد `sigsetjmp` رو کال می‌کنه که باعث میشه پوینتر به استک و پوینتر اینستراکشن و چندتا رجیستر تو یه بافری ذخیره شن. 
	2. تو خط ۴۵۱۶ اون رو میده به بافر گلوبال `PG_exception_stack` و از این به بعد، داخل این پروسه، اگه از ماکروی `PG_TRY/PG_END_TRY` از سرآیند `src/include/utils/elog.h` هر جا داخل هر فانکشنی استفاده شده باشه و اکسپشنی رخ بده، در نهایت بر می‌گردیم به خط ۴۴۰۲ این فایل دوباره.
8. و در نهایت از خط ۴۵۲۵ لوپ بی‌نهایت‌مون ران میشه و می‌رسه به ۴۷۰۷ و بلاک میشه تا زمانی که یه دستور از کلاینت به سرور برسه.
### کوئری سلکت
حالا فرض کنیم می‌خوایم یه کوئری سلکت ساده بزنیم. کوئری رو داخل کنسول یا ابزارمون می‌نویسیم و اینتر رو می‌زنیم. در ادامه‌ی فایل `postgres.c` اتفاقی که داخل سرور می‌افته اینه:
1. در خط ۴۷۰۷ کوئری خونده میشه و قبل از رفتن به مرحله‌ی بعد، اگه این وسط دستور ریلود کانفیگ اومده باشه، کانفیگ پروسه‌مون بروز میشه.
2. در خط ۴۷۵۷ می‌ریم سراغ تصمیم گرفتن برای کاری که کلاینت گفته انجام بدیم. این کار داخل پروتکل پسگره با اولین کاراکتر مشخص میشه.
3. چون ما خواستیم یه کوئری ساده بزنیم، گزینه‌ی `PqMsg_Query` انتخاب می‌شه و می‌پریم به خط ۱۰۱۷ و تابع `exec_simple_query` ران میشه: 
	1. این تابع در خط ۱۰۵۱ یه ترنزاکشن باز می‌کنه
	2. در خط ۱۰۷۰ کوئری رو پارس می‌کنه که با روشن کردن فلگ‌های دیباگی که داخل کد هست میشه دیدشون. اگه با `pg_ctl` ران کرده باشیم مثلاً اینطوری:
	   ```shell
	   /usr/local/pgsql/bin/pg_ctl -D myhmdb -l logfile -o "-c debug_print_raw_parse=1 -c debug_print_rewritten=1 -c debug_print_plan=1" start
	   ```
	   تنها نکته اینکه حاصل پارس کوئری در مرحله‌ی قبل یه لیسته. به ازای هر کوئری که با سمی‌کالن جدا کرده باشیم، یدونه درخت داریم. اینجا چون ما فرض کردیم می‌خوایم یدونه کوئری سلکت بزنیم، پس میشه ۱ درخت که `SELECTSTMT` هست.
	3. در خط ۱۱۰۰ می‌رسیم به یه حلقه روی هر کدوم از درختا داخل لیست‌مون. برای هر درخت:
		1. در خط ۱۱۲۴ یه تگ می‌گیریم. از این تگ برای اسم گذاشتن روی پروسه و اینطور چیزا استفاده می‌کنیم. لیست همه‌ی تگ‌ها رو میشه در فایل `src/backend/tcop/utility.c` خط ۲۳۶۸ دید.
		2. در خط ۱۱۶۱ چک می‌کنیم اگه احتمالاً سیگنال اینتراپت گرفتیم، بریم تابع `ProcessInterrupts` رو در خط ۳۳۰۴ اجرا کنیم ببینیم چه خبره. اگه ماجرا بگا رفته که همین جا تموم کنیم ران کوئری رو.
		3. در خط ۱۱۹۵ کوئری رو در صورت نیاز بازنویسی می‌کنه (که با توجه به کوئری سلکت ما در اینجا `commandType` مقدار یک رو می‌گیره ) و در خط ۱۱۹۸ یه لیست از کوئری پلن‌/ها می‌سازه (که پلن‌مون در اینجا `SEQSCAN` میشه). سورس پلنر رو میشه در `src/backend/optimizer/plan/planner.c` دید.
		4. در خط ۱۲۱۵ دوباره مثل آیتم ۲ وجود سیگنال اینتراپت رو چک می‌کنیم.
		5. و در خط ۱۲۲۱ می‌رسیم به ساختن پرتال. پرتال چیزیه که وضعیت یه کوئری رو از نظر کرسر، ترانزاکشن، اسنپ‌شات، مموری و وضعیت اجرا مدیریت می‌کنه. 
		6. بعد از ساختن پرتال، در خط ۱۲۴۰ `PortalStart` صدا زده میشه. با کال شدن این تابع در خط ۴۹۱ ما استراکت `QueryDesc` رو می‌سازیم و در خط ۵۱۳ تابع `ExecutorStart` صدا زده میشه که به هر دوی این عزیزان در ادامه بر می‌گردیم، اولی برای ساختار و دومی برای گرفتن مسیر  ساختن ام خودمون.
		7. از خط ۱۲۴۸ تا ۱۲۶۲ فرمت خروجی (متن یا باینری) مشخص می‌شه.
		8. در خط ۱۲۶۹ پرتال رو به سوکت کلاینت می‌دیم. در واقع در فایل `src/backend/access/common/printtup.c` خط ۳۰۵ فانکشن `printtup` چیزی هست که تاپل‌های کوئری رو تبدیل به خروجی می‌کنه و جای خوبیه برای فهمیدن چگونگی استفاده از تاپل‌ها که بعد می‌تونیم بهش برگردیم.
		9. در خط ۱۲۷۹ پرتال‌مون رو با صدا زدن تابع `PortalRun` ران می‌کنیم و منتظر تموم شدن کار می‌مونیم. این جاییه که بهش بر می‌گردیم که ببینیم از ام‌مون چطور داره استفاده میشه.
		10. در پایان حلقه هم که ترانزکشن رو می‌بندیم و بر می‌گردیم اول حلقه اگه کوئری دیگه‌ای داشتیم.
	4. و در خط ۱۳۵۴ هم ترنزاکشنی که باز کردیم بسته میشه.
4. می‌ریم به آیتم ۸ لیست قبلی.

احتمالاً کاربردی‌ترین چیزی که از این دو بخش می‌تونیم بفهمیم هزینه‌ی ایجاد یه کانکشن جدید به پسگره‌ست و دیدیم که در عمل یه تریدآفه بین منابع سروری که پسگره روش بالا میاد و هم‌روندی ابزاری که داریم توسعه می‌دیم.
### کوئری دسکریپتور
در آیتم ۳.۳.۶ بخش قبل گفتیم که استراکت `QueryDesc` با صدا زدن تابع `CreateQueryDesc` در خط ۴۹۱ ساخته میشه. این استراکت تمام چیزی که برای اجرای کوئری نیاز هست رو در داخل خودش داره:
```c
typedef struct QueryDesc  
{  
    /* These fields are provided by CreateQueryDesc */  
    CmdType       operation;    /* CMD_SELECT, CMD_UPDATE, etc. */  
    PlannedStmt *plannedstmt;  /* planner's output (could be utility, too) */  
    const char *sourceText;       /* source text of the query */  
    Snapshot   snapshot;     /* snapshot to use for query */  
    Snapshot   crosscheck_snapshot;   /* crosscheck for RI update/delete */  
    DestReceiver *dest;          /* the destination for tuple output */  
    ParamListInfo params;     /* param values being passed in */  
    QueryEnvironment *queryEnv; /* query environment passed in */  
    int          instrument_options; /* OR of InstrumentOption flags */  
  
    /* These fields are set by ExecutorStart */    
    TupleDesc  tupDesc;      /* descriptor for result tuples */  
    EState    *estate;          /* executor's query-wide state */  
    PlanState  *planstate;    /* tree of per-plan-node state */  
  
    /* This field is set by ExecutePlan */    
    bool      already_executed;  /* true if previously executed */  
  
    /* This is always set NULL by the core system, but plugins can change it */    struct Instrumentation *totaltime; /* total time spent in ExecutorRun */  
}
```
سورس این استراکت رو میشه در `src/include/executor/execdesc.h` دید. برای این که گوشی هم دست‌مون بیاد، با توجه به کوئری سلکتی که هدف داریم بزنیم، مقدار `operation` میشه `CMD_SELECT`، مقدار `plannedstmt`  شد درختی که ریشه‌اش `SEQSCAN`عه. اسنپ‌شات‌ها رو که مفصل بهشون بر می‌گردیم. `dest` که مقدارش میشه `DestRemote` با توجه به اینکه از کنسول داریم کوئری می‌زنیم به سرور. `EState` هم وضعیت Executer رو نگه می‌داره که با همسایه‌هاش فعلن خالی‌ن.
### استارت پرتال
در ادامه‌ی اجرای کوئری سلکت‌مون، دیدیم که استارت پرتال اونجاییه که عملاً وارد ساختن پلن اجرایی میشیم و می‌رسه به ساختن اکسس متدی که داریم توسعه میدیم. در این بخش هم قراره مثل دو بخش قبل مسیر اجرای کوئری سلکت‌مون رو دنبال کنیم.
1. در ادامه‌ی اجرا از آیتم ۳.۳.۶ لیست بخش کوئری سلکت، در بالا دیدیم که در خط ۱۲۴۰ از فایل `src/backend/tcop/postgres.c` تابع `PortalStart` کال میشه.
2. در خط ۴۲۹ تابع `PortalStart` در فایل `src/backend/tcop/pquery.c` تعریف شده.  این تابع یکی از جاهاییه که می‌تونیم ساختار `PG_TRY/PG_CATCH/PG_END_TRY` که در آیتم ۷.۲ زیربخش اول بهش اشاره کردیم رو ببینیم.
3. در خط ۵۱۳ تابع `ExecutorStart` از فایل `src/backend/executor/execMain.c` تعریف شده در خط ۱۲۲ ران میشه و از اونجا تابع `standard_ExecutorStart` تعریف شده در خط ۱۴۱ ران میشه.
4. داخل تابع `standard_ExecutorStart` تابع میاد فیلدهای `estate` که مرتبط با استیت خودش هست رو پر می‌کنه و در خط ۲۶۱ `InitPlan` رو ران می‌کنه.
5. تابع `InitPlan` دوتا پارامتر می‌گیره که یکیش `QueryDesc`عه و دومی فلگ‌های مرتبط با اجرا. داخل این تابع اول پرمیژن مرمیژنا چک می‌شه و در نهایت می‌رسه به خط ۹۸۷ که صدا زدن تابع `ExecInitNode` باشه (همونطور که یادمونه، برای کوئری سلکت ما، با تایپ `SEQSCAN` طرف بودیم).
6. داخل فایل `src/backend/executor/execProcnode.c` و سوئیچ کیس تابع `ExecInitNode` تابع `ExecInitSeqScan` کال میشه و میریم به فایل `src/backend/executor/nodeSeqscan.c`.
7. داخل تابع `ExecInitSeqScan` قراره یه استراکت `SeqScanState` برگردونیم که قبل از ادامه بهتره بشناسیمش:
	1. این استراکت یه زیراستراکت `PlanState` در فایل `src/include/nodes/execnodes.h` خط ۱۱۵۹عه که مثل همون پلنه که تا حالا داشتیم ولی داره شیوه‌ی اجرا رو فراهم می‌کنه. همه‌ی پلن‌های اجرائی این استراکت رو به ارث می‌برن.
	2. استراکت `SeqScanState` دوتا فیلد بیشتر از `PlanState` داره: `ScanState` و `Size`. دومی رو نمی‌دونم چیه هنوز.
	3. تایپ `ScanState` فیلد `PlanState` رو داره که گفتیم چیه، `Relation` رو داره که جدول‌مون رو مشخص می‌کنه (و بهش بر می‌گردیم در ادامه :دی)، `TableScanDescData` و `TupleTableSlot` رو داره که داخلش در نهایت فیلد با تایپ `TupleTableSlotOps` رو داره که داخل بخش اسلات دیدیم چیه.
8. تابع همونطور که از اسمش برمیاد داره یه سری فیلد رو پر می‌کنه تا خط ۲۴۰ که `ExecOpenScanRelation` و داره جدولی که می‌خوایم اسکن کنیم رو باز می‌کنه.
9. تابع `ExecOpenScanRelation` در `src/backend/executor/execUtils.c`  خط ۷۴۲ قرار داره و تابع `ExecGetRangeTableRelation` رو از همین فایل خط ۸۲۵ کال می‌کنه.
	1. تابع `ExecGetRangeTableRelation` اول تلاش می‌کنه ریلیشنی که می‌خوایم رو از کش بخونه. ما فرض می‌کنیم داخل کش نداریمش و خط ۸۵۱ تابع `table_open`  رو کال می‌کنیم. این تابع در خط ۴۰ فایل `src/backend/access/table/table.c` قرار داره و تابع `relation_open` رو کال می‌کنه که داخل فایل `src/backend/access/common/relation.c` خط ۴۷ قرار داره.
	2. داخل `relation_open` در خط ۵۸ تابع `RelationIdGetRelation` از فایل `src/backend/utils/cache/relcache.c` خط ۲۰۹۹ کال میشه. این تابع خود خود کشه :)) و ما باز فرض می‌کنیم کش نداریم چیزی.
	3. با کش نداشتن خط ۲۱۴۲ تابع `RelationBuildDesc`  کال میشه. این تابع در خط ۱۰۵۹ فایل `src/backend/utils/cache/relcache.c` قرار داره. این تابع در خط ۱۱۱۳ تابع `ScanPgRelation` رو کال می‌کنه که مثل زدن کوئری
	    ```sql
	    select *  
		from pg_catalog.pg_class;
	    ```
	    روی دیتابیس می‌مونه. در واقع ما داریم از خود دیتابیس و هیپ استفاده می‌کنیم برای ساختن هر چیز دیگه‌ای که داخل پسگره هست. مثل وقتی که از گو برای نوشتن خود گو استفاده کردن. خیلی خفنه خدایی :)). در واقع تابع قراره استراکت `Form_pg_class` برگردونه که در خط ۳۲ فایل `src/include/catalog/pg_class.h` با ماکرو تعریف شده. اگه فیلدهای این استراکت رو ببینیم نعل به نعل کپی همون ستون‌های جدولیه که گفتیم. و این عجیب نیست. گفتیم که تاپل روی دیسک و مموری ساختار شبیه هم دارن و این هم از خفنیه سی‌ه و هم داداش‌مون که پسگره رو نوشته. به صورت مشخص با استفاده از تابع درون‌خطی خط ۷۲۹ فایل `src/include/access/htup_details.h`:
	    ```c
	    /*  
		 * GETSTRUCT - given a HeapTuple pointer, return address of the user data 
		 */
		static inline void *  
		GETSTRUCT(const HeapTupleData *tuple)  
		{  
		    return ((char *) (tuple->t_data) + tuple->t_data->t_hoff);  
		}
	    ```
	    واقعاً چقد قشنگه :)). در ادامه‌ی این مسیر اگه به تابع `ScanPgRelation` نگاهی بندازیم، یه نمونه از کوئری زدن خیلی خام با کد رو می‌تونیم ببینیم که الان مهم نیست.
	4. در ادامه در خط ۱۲۳۰، تابع `RelationInitTableAccessMethod` (واقع در خط ۱۸۲۹) کال میشه و از اونجا تابع `InitTableAmRoutine` در خط ۱۸۲۰ کال میشه که تابع `GetTableAmRoutine` در فایل `src/backend/access/table/tableamapi.c` خط ۲۸ کال شه :)) که اکسس‌متدی که موقع تعریف جدول داشتیم به ریلیشن‌مون الصاق میشه. 
	    خیلی مفصله ماجرای لود کردن فانکشن سی ولی در نهایت به تابع `load_external_function` از فایل `src/backend/utils/fmgr/dfmgr.c` خط ۹۵ می‌رسیم. خیلی خلاصه ماکروی `PG_FUNCTION_INFO_V1` که بالای تابع در ام‌مون قرار دادیم، اینطور چیزی جنریت می‌کنه:
	    ```c
	    extern __attribute__((visibility("default"))) Datum hmam_tableam_handler(FunctionCallInfo fcinfo);
	
		extern __attribute__((visibility("default"))) const Pg_finfo_record *pg_finfo_hmam_tableam_handler(void);
	
		const Pg_finfo_record *pg_finfo_hmam_tableam_handler(void) {
		    static const Pg_finfo_record my_finfo = {1};
		    return &my_finfo;
		}
	    ```
	    اولی که اکسترن همون فانکشن خودمونه آخر فایل ام و دوتا خط بعدی هم برای چک کردن ورژن و ولیدیشن استفاده شدن (احتمالاً برای اینکه بعداً اگه خواستن یه ورژن ۲ بدن روی ساختن فانکشن سی داخل پسگره). کار نداریم، خلاصه میاد داخل کد تاپلی که هندلرمون رو اول کار ساختیم (کوئری `CREATE FUNCTION`) رو می‌گیره و با استفاده از ماکروی `TextDatumGetCString` تبدیلش می‌کنه به استرینگ سی (آرایه کاراکتر با ترمینیشن نال) و از تابع `dlsym` داخل سی (که منوالش رو میشه با `man 3 dlsm` دید) استفاده می‌کنه و یه پوینتر به فانکشن‌مون (در حقیقت یه `void *` بر می‌گرده) بر می‌گردونه و در نهایت با کست `(TableAmRoutine *)` تبدیل میشه به چیزی که ما نوشتیم :)).
10. در خط ۲۷۱ هم تابع `ExecSeqScan` به عنوان تابعی که قراره از ام استفاده کنه و دیتا رو بکشه بیرون اساین میشه.
11. در نهایت ما بر می‌گردیم به تابع `PortalStart` و خط ۶۰۵ که وضعیت پرتال رو به `PORTAL_READY` یعنی آماده‌ی ران تغییر می‌ده.
### اجرای پرتال
الان به جایی رسیدیم که پلن اجرایی رو داریم و میریم برای اجرا.
1. در ادامه‌ی اجرا از آیتم ۳.۳.۹ لیست بخش کوئری سلکت در بالا، از فایل `src/backend/tcop/pquery.c`  خط ۶۸۰ تابع `PortalRun` ران میشه.
2. در خط ۷۱۱ وضعیت پرتال به `PORTAL_ACTIVE` تغییر پیدا می‌کنه و در خط ۷۶۰ تابع `PortalRunSelect` ران میشه و از اونجا در خط ۹۱۶ تابع `ExecutorRun` ران میشه.
3. داخل فایل `src/backend/executor/execMain.c` در خط ۳۰۳ تابع `standard_ExecutorRun` ران میشه که این تابع در خط ۳۰۷ تعریف شده.
4. داخل تابع `standard_ExecutorRun` تابع میاد در خط ۳۵۱ اطلاعات هدر ریزالت نهایی رو برای کلاینت می‌فرسته. این اطلاعات شامل تعداد اتریبیوت‌های تاپل و اسم هر اتریبیوت به همراه تایپ‌شونه.
5. در نهایت تابع `ExecutePlan` در خط ۳۶۶ کال میشه و ما وارد فاز اجرای پلن میشیم.
6. تابع `ExecutePlan` در خط ۱۶۵۶ فایل `src/backend/executor/execMain.c` تعریف شده. از خط ۱۶۹۹ ما وارد یک لوپ بی‌نهایت میشیم که:
	1. در خط ۱۷۰۷ پلن ران میشه و اسلات متناظر باهاش بر می‌گرده. این فرآیند اینطور طی میشه:
		1. تابع `ExecProcNode` در خط ۱۷۰۷ کال میشه. این تابع در سرآیند `src/include/executor/executor.h` تعریف شده.
		2. در تابع `ExecProcNode` تابع `ExecProcNode` تعریف شده به عنوان یه پوینتر به فانکشن داخل نودمون هست. نود ما همونطور که در آیتم ۷ لیست بخش استارت پرتال دیدم، `SeqScanState` هست و مقدار هم که بهش دادیم `ExecSeqScan` بود (آیتم ۱۰ لیست قبل) که بریم ببینم چیکار میکنه:
			1. تابع در فایل `src/backend/executor/nodeSeqscan.c` خط ۱۰۹ تعریف شده و به عنوان مقدار برگشتی، تابع `ExecScanExtended` رو کال کرده که پارامتر `accessMtd`ش تابع `SeqNext` هست.
			2. تابع `ExecScanExtended` در سرآیند `src/include/executor/execScan.h` خط ۱۵۹ تعریف شده. این تابع اگه بروجکشن (سلکت ستون) یا شرط نداشته باشه به سادگی تابع `SeqNext` رو روی جدول کال می‌کنه و بر می‌گرده. اگه باشه داخل یه حلقه بی‌نهایت اونقدر می‌گرده که یا دیگه چیزی نباشه یا به چیزی که می‌خواد برسه و پروجکت کنه و برگرده.
			3. تابع `SeqNext` در فایل `src/backend/executor/nodeSeqscan.c` خط ۵۰ تعریف شده و رفتارش اینطوریه:
				1. اگه از قبل داخل `SeqScanState` ما `TableScanDescData` رو نداشته باشیم، میاد تابع `table_beginscan` از سرآیند `src/include/access/tableam.h` خط ۸۷۶ رو کال می‌کنه. این تابع هم از اکسس متدی که ما داریم پیاده‌سازی می‌کنیم، `scan_begin` رو کال می‌کنه. در نهایت مقدار رو به ‍`TableScanDescData` اساین می‌کنه.
				2. در ادامه، تابع `table_scan_getnextslot` از همون سرآیند خط ۱۰۱۹ رو کال می‌کنه و این تابع میاد تابع `scan_getnextslot` ام ما رو کال می‌کنه. اگه اوکی باشه اسلاتی که داخل این تابع پر میشه بر می‌گرده و در غیراینصورت `NULL` بر می‌گرده.
	2. در خط ۱۷۱۳ چک میشه که اسلات اگه نال باشه دیگه چیزی برای فچ کردن نیست و تمومه کار.
	3. در خط ۱۷۳۸ ریزالت رو برای کلاینت می‌فرسته و اگه نتونه این کارو بکنه از ادامه‌ی لوپ منصرف میشه.
	4. در خط ۱۷۵۶ اگه به تعدادی که می‌خواستیم تاپل گرفتیم خارج میشه.
7. در نهایت کار اجرا تمومه.

عالی. پس تا اینجا ما دل و روده‌ی پسگره رو ریختیم بیرون و فهمیدیم چطور بالا میاد. وقتی کانکشن زده می‌شه بهش چه اتفاقی می‌افته، چطوری کوئری ما تبدیل به پلن میشه و در نهایت چطور از متدهایی که ما داریم برای اکسس متدمون پیاده می‌کنیم استفاده می‌شه داخل اون پلن و در نهایت چطور جواب به کلاینت بر می‌گرده. عالی، خیلی هم عالی.
## ادامه‌ی اولین نمایش 
### خلوت کردن `hmam_scan_begin`
در بالاتر وقتی می‌خواستیم فانکشن `hmam_scan_begin` صرفاً راه بیفته کد هیپ رو کپی کردیم و ازش استفاده کردیم. الان که یذره عمقی‌تر فهمیدیم ماجرا چیه به نظر نیازی بهش نداریم و صرفاً میشه با خود تایپ اصلی پیش رفت. پس قبل از رفتن سر وقت `hmam_scan_getnextslot` این فانکشن رو خلوت می‌کنیم:
```diff
hmam_scan_begin(Relation rel,
                 ParallelTableScanDesc pscan,
                 uint32 flags) {
     flog;
-    HeapScanDesc scan = (HeapScanDesc) palloc_object(HeapScanDescData);
-
-    scan->rs_base.rs_rd = rel;
-    scan->rs_base.rs_snapshot = snapshot;
-    scan->rs_base.rs_nkeys = nkeys;
-    scan->rs_base.rs_flags = flags;
-    scan->rs_base.rs_parallel = NULL;
-    scan->rs_strategy = NULL;
-    scan->rs_cbuf = InvalidBuffer;
-
-    if (scan->rs_base.rs_flags & (SO_TYPE_SEQSCAN | SO_TYPE_SAMPLESCAN)) {
-        Assert(snapshot);
-        PredicateLockRelation(rel, snapshot);
-    }
-
-    scan->rs_ctup.t_tableOid = RelationGetRelid(rel);
-    scan->rs_parallelworkerdata = NULL;
-
-    if (nkeys > 0)
-        scan->rs_base.rs_key = palloc_array(ScanKeyData, nkeys);
-    else
-        scan->rs_base.rs_key = NULL;
+    TableScanDesc scan = palloc_object(TableScanDescData);

-    scan->rs_read_stream = NULL;
+    scan->rs_rd = rel;
+    scan->rs_snapshot = snapshot;
+    scan->rs_nkeys = nkeys;
+    scan->rs_key = key;
+    scan->rs_parallel = pscan;
+    scan->rs_flags = flags;

-    return (TableScanDesc) scan;
+    return scan;
 }
```

### پیاده‌سازی `hmam_scan_getnextslot`
الان می‌دونیم که `TableScanDesc` که تو امضای تابع هست، همونیه که تو متد بالا پاس دادیم. چیزی که مشخصه اینه که این تابع تا زمانی که به آخر کار نرسیدیم مدام کال می‌شه و بعد ما باید شکلی از مدیریت کرسر (که تا کجا خوندیم) رو هم به `TableScanDesc` اضافه کنیم، ولی برای الان صرفاً‌ می‌خوایم یچی بگیریم و نشون بدیم، پس کافیه آیدی جدول رو بگیرم، بدیم به هش‌مپ‌مون، یدونه مقدار بگیریم و به عنوان تاپل بدیم به اسلات‌مون.
```c

```