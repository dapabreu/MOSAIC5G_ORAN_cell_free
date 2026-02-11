# Intro

💡**Pilot signals are used to estimate the channel characteristics, such as amplitude and phase distortions, allowing the receiver to compensate for these distortions and accurately decode the transmitted data.**

**Importance:** Pilot signals play a crucial role in channel estimation as they provide a reference for the receiver to measure the channel's effects on the transmitted signal. Accurate channel estimation is essential for reliable data transfer, especially in environments with significant multipath fading and interference.

# Types of Pilot Signals

In NR, there are new reference signals: 

- Phase Tracking Reference Signal
- PBCH Reference Signal
- Time/Frequency Tracking Reference Signal
- DMRS (Demodulation Reference Signal) for various channels(e.g, PDCCH, PDSCH etc)

| Reference Signal | Functionality                         | 3GPP Reference                      |
|------------------|---------------------------------------|--------------------------------------|
| **SSB**          | Required for Sync Detection           | 38.211-7.4.3.1, 38.213-4.1           |
| **CSI-RS**       | CSI acquisition, Beam Management      | 38.211-7.4.1.5                       |
| **PDSCH DMRS**   | Required for PDSCH Demodulation       | 38.211-7.4.1.1                       |
| **PUSCH DMRS**   | Required for PUSCH Demodulation       | 38.211-6.4.1.1                       |
| **PDCCH DMRS**   | Required for PDCCH Demodulation       | 38.211-7.4.1.3                       |
| **PUCCH DMRS**   | Required for PUCCH Demodulation       | 38.211-6.4.1.3                       |
| **SRS**          | Sounding Reference Signal             | 38.211-6.4.1.4                       |
| **PBCH DMRS**    | Required for PBCH Demodulation        | 38.211-7.4.1.4                       |
| **PTRS**         | Used for Phase Tracking for PDSCH     | 38.211-7.4.1.2                       |
| **TRS**          | Used for Time Tracking                | 38.211-7.4.1.6                       |


# Types found in OAI

In the `NR_REFSIG` folder we find several types of files related to reference signals:

- `dmrs_nr.c`: generation of drms sequences
- `nr_drms_rx.c`: top level routines for generating DMRS from 38.211
- `nr_gen_mod_table.c`: calculate and store the constellations of modulation processes (QAM, QSPK, etc.)
- `nr_gold_ue.c`: responsible for generating the Gold sequences for the pilot signals (DMRS) used in the side channel (Sidelink),
- `nr_refsig_common.h`: defines the functions (this is a header file) for generating the gold sequences used for reference signals.
- The functions mentioned are:

```C
uint32_t *gold_cache(uint32_t key, int length);
uint32_t *nr_gold_pbch(int Lmax, int Nid, int n_hf, int ssb);
uint32_t *nr_gold_pdsch(int N_RB_DL, int symbols_per_slot, int nid, int nscid, int slot, int symbol);
uint32_t *nr_gold_pusch(int N_RB_UL, int symbols_per_slot, int Nid, int nscid, int slot, int symbol);
uint32_t *nr_gold_csi_rs(int N_RB_DL, int symbols_per_slot, int slot, int symb, uint32_t Nid);
uint32_t *nr_gold_prs(int nid, int slot, int symbol);
```
    
- `pss_nr.h`: definições para o Primary Synchronization Signal
- `ptrs_nr.c`: phase tracking reference signals, e elementos relacionados
- `refsig.c`: é onde estão implementadas as funções de geração das sequências Gold para todos os sinais de referência NR, vindas do `nr_refsig_common.h`.
- `ss_pbch_nr.h`: define elements related to SS/PBCH block ie synchronisation (pss/sss) and pbch
- `sss_nr.h` : define elements related to the Secondary Synchronization Signals
- `ul_ref_seq_nr.c`: function to generate uplink reference sequences

# Generation of Reference Signals

💡 To generate Reference Signals, something called: **Gold Code** or **Gold Sequence.** is used.

## What are Gold Sequences/Gold Codes

A Gold Code is a *pseudorandom bit sequence*, i.e., a sequence of bits that *looks* random, but is reproducible - **from an initial (know) state, that we call `seed`.**

They are used for:

- Data bit scrambling
- Pilot signal modulation
- Reference signal generation for channel measurement


At a high level, they have to do with finding and/or picking a particular signal out of the "noise" (which would include other competing signals as well) - especialmente em ambientes com alta densidade.

What makes Gold codes particularly useful compared to other sequences in general is that:

> Gold codes have bounded small cross-correlations within a set, which  is useful when multiple devices are broadcasting in the same frequency  range - **one of the most useful properties is this low correlation** i.e. a set of Gold codes are good at not interfering with **each other**

So if hundreds of cell phones or dozens of GPS satellites are all using the same general Gold code *system* but with a different *key* for each it is (relatively) easy to separate the signals for all of  those different keys individually — even when they're all transmitting their individual patterns simultaneously.

### How are gold codes generated?

Using the following method (theoretically):

- We have two LFSRs: Linear Feedback Shift Register
- We combine the two circuits with an XOR
- We produce a bit sequence that takes a long time to repeat and also has excellent low-correlation properties.

**The choice of the initial state (seed) depends on parameters such as cell ID, slot, and user RNTI.**

# How Gold Codes are generated in OAI

For each of these functions, which generate the Gold Codes for each pilot signal, we have:

```C
uint32_t *gold_cache(uint32_t key, int length);
uint32_t *nr_gold_pbch(int Lmax, int Nid, int n_hf, int ssb);
uint32_t *nr_gold_pdsch(int N_RB_DL, int symbols_per_slot, unsigned short n_idDMRS, int ns, int l);
uint32_t *nr_gold_pdcch(int N_RB_DL, int symbols_per_slot, int nid, int nscid, int slot, int symbol);
uint32_t *nr_gold_pusch(int N_RB_UL, int symbols_per_slot, int Nid, int nscid, int slot, int symbol);
uint32_t *nr_gold_csi_rs(int N_RB_DL, int symbols_per_slot, int slot, int symb, uint32_t Nid);
uint32_t *nr_gold_prs(int nid, int slot, int symbol);
```

**List of Variables:** 

| Parameter | Meaning |
| --- | --- |
| `N_RB_DL` / `UL` | Number of resource blocks (bandwidth in RBs) |
| `symbols_per_slot` | Number of OFDM symbols per slot |
| `slot` / `ns` | Slot index within the frame |
| `symbol` / `l` | Index of the OFDM symbol within the slot |
| `nid` / `Nid` | Physical Cell ID (PCI) |
| `nscid` | Scrambling secondary identity |
| `n_idDMRS` | Reference signal ID |
| `n_hf` | Half-frame number |
| `ssb` | SS Block index |
| `Lmax` | Max. number of SS Blocks per half-frame |

*Example:*

```c
uint32_t *nr_gold_pbch(int Lmax, int Nid, int n_hf, int l)
{
  int i_ssb = l & (Lmax - 1);
  int i_ssb2 = i_ssb + (n_hf << 2);
  uint32_t x2 = (1 << 11) * (i_ssb2 + 1) * ((Nid >> 2) + 1) + (1 << 6) * (i_ssb2 + 1) + (Nid & 3);
  return gold_cache(x2, NR_PBCH_DMRS_LENGTH_DWORD);
}
```

Calling the function `gold_cache():`

1. Traverse the cache table.
2. If you find a sequence with the same `key` and the same suitable **length**, you reuse the sequence.
3. If you find one with the same `key` but a shorter length, you force a reordering to replace it.
4. If none exists, create a new entry with `x1 = 0`, `x2=key`, and call `lte_gold_generic()` to create a gold sequence.


💡 **That is, the same UE can have the same sequence, but different UEs cannot.**

Attention **To this important point:**

- When we call: 
- `gold_cache(x2, NR_PBCH_DMRS_LENGTH_DWORD)`, lembrar que o `x2 =   uint32_t x2 = (1 << 11) * (i_ssb2 + 1) * ((Nid >> 2) + 1) + (1 << 6) * (i_ssb2 + 1) + (Nid & 3);`

This expression may very but we know that `x2` will be different for every `nid`, `slot`, `simbolo`, `nscid`, etc.
- **THIS GUARANTEES THE UNIQUENESS OF THE KEY, IN DIFFERENT CONTEXTS!**

```c

uint32_t *gold_cache(uint32_t key, int length)
{
  (void)pthread_once(&gold_key_once, make_table_key);
  gold_cache_table_t *tableCache;
  if ((tableCache = pthread_getspecific(gold_table_key)) == NULL) {
    tableCache = calloc(1, sizeof(gold_cache_table_t));
    (void)pthread_setspecific(gold_table_key, tableCache);
  }

  // align for AVX512
  length = ((length + grain - 1) / grain) * grain;
  tableCache->calls++;

  // periodic refresh
  if (tableCache->calls > REFRESH_RATE)
    refresh_table(tableCache, 0);

  uint32_t *ptr = tableCache->table;
  // check if already cached
  for (; ptr < tableCache->table + tableCache->tblSz; ptr += roundedHeaderSz) {
    gold_cache_t *tbl = (gold_cache_t *)ptr;
    tableCache->iterate++;
    if (tbl->length >= length && tbl->key == key) {
      tbl->usage++;
      return ptr + roundedHeaderSz;
    }
    if (tbl->key == key) {
      // We use a longer sequence, same key
      // let's delete the shorter and force reorganize
      tbl->usage = 0;
      tableCache->calls += REFRESH_RATE;
    }
    if (!tbl->length)
      break;
    ptr += tbl->length;
  }

  // not enough space in the table
  if (!ptr || ptr > tableCache->table + tableCache->tblSz - (2 * roundedHeaderSz + length))
    refresh_table(tableCache, 2 * roundedHeaderSz + length);

  // We will add a new entry
  uint32_t *firstFree;
  int size = 0;
  for (firstFree = tableCache->table; firstFree < tableCache->table + tableCache->tblSz; firstFree += roundedHeaderSz) {
    gold_cache_t *tbl = (gold_cache_t *)firstFree;
    if (!tbl->length)
      break;
    firstFree += tbl->length;
    size++;
  }
  if (!tableCache->calls)
    LOG_D(PHY, "Number of entries (after reorganization) in gold cache: %d\n", size);

  gold_cache_t *new = (gold_cache_t *)firstFree;
  *new = (gold_cache_t){.key = key, .length = length, .usage = 1};
  unsigned int x1 = 0, x2 = key;
  uint32_t *sequence = firstFree + roundedHeaderSz;
  *sequence++ = lte_gold_generic(&x1, &x2, 1);
  for (int n = 1; n < length; n++)
    *sequence++ = lte_gold_generic(&x1, &x2, 0);
  LOG_D(PHY, "created a gold sequence, start %d; len %d\n", key, length);
  return firstFree + roundedHeaderSz;
}
```
In the `lte_gold_generic()` function:

- Here, the **Gold Code** will be generated. And the `X2` will be generated taking into account unique parameters of the context in which it is located

```c
static inline  uint32_t lte_gold_generic(uint32_t *x1, uint32_t *x2, uint8_t reset)
{
  int32_t n;

  // 3GPP 3x.211
  // Nc = 1600
  // c(n)     = [x1(n+Nc) + x2(n+Nc)]mod2
  // x1(n+31) = [x1(n+3)                     + x1(n)]mod2
  // x2(n+31) = [x2(n+3) + x2(n+2) + x2(n+1) + x2(n)]mod2
  if (reset)
  {
      // Init value for x1: x1(0) = 1, x1(n) = 0, n=1,2,...,30
      // x1(31) = [x1(3) + x1(0)]mod2 = 1
      *x1 = 1 + (1U<<31);
      // Init value for x2: cinit = sum_{i=0}^30 x2*2^i
      // x2(31) = [x2(3)    + x2(2)    + x2(1)    + x2(0)]mod2
      //        =  (*x2>>3) ^ (*x2>>2) + (*x2>>1) + *x2
      *x2 = *x2 ^ ((*x2 ^ (*x2>>1) ^ (*x2>>2) ^ (*x2>>3))<<31);

      // x1 and x2 contain bits n = 0,1,...,31

      // Nc = 1600 bits are skipped at the beginning
      // i.e., 1600 / 32 = 50 32bit words

      for (n = 1; n < 50; n++)
      {
          // Compute x1(0),...,x1(27)
          *x1 = (*x1>>1) ^ (*x1>>4);
          // Compute x1(28),..,x1(31) and xor
          *x1 = *x1 ^ (*x1<<31) ^ (*x1<<28);
          // Compute x2(0),...,x2(27)
          *x2 = (*x2>>1) ^ (*x2>>2) ^ (*x2>>3) ^ (*x2>>4);
          // Compute x2(28),..,x2(31) and xor
          *x2 = *x2 ^ (*x2<<31) ^ (*x2<<30) ^ (*x2<<29) ^ (*x2<<28);
      }
  }

  *x1 = (*x1>>1) ^ (*x1>>4);
  *x1 = *x1 ^ (*x1<<31) ^ (*x1<<28);
  *x2 = (*x2>>1) ^ (*x2>>2) ^ (*x2>>3) ^ (*x2>>4);
  *x2 = *x2 ^ (*x2<<31) ^ (*x2<<30) ^ (*x2<<29) ^ (*x2<<28);

  // c(n) = [x1(n+Nc) + x2(n+Nc)]mod2
  return(*x1^*x2);
}
```

At a higher level, for example, when we are doing **PBCH channel estimation,** we use `gold_pbch` as follows:

```c

int nr_pbch_channel_estimation(const NR_DL_FRAME_PARMS *fp,
                               const sl_nr_ue_phy_params_t *sl_phy_params,
                               int estimateSz,
                               struct complex16 dl_ch_estimates[][estimateSz],
                               struct complex16 dl_ch_estimates_time[][fp->ofdm_symbol_size],
                               const UE_nr_rxtx_proc_t *proc,
                               unsigned char symbol,
                               int dmrss,
                               uint ssb_index,
                               uint n_hf,
                               int ssb_start_subcarrier,
                               const c16_t rxdataF[][fp->samples_per_slot_wCP],
                               bool sidelink,
                               uint Nid)
{

[....]
[....]
[....]

    nushift = Nid % 4;

    AssertFatal(dmrss >= 0 && dmrss < 3, "symbol %d is illegal for PBCH DM-RS \n", dmrss);

    gold_seq = nr_gold_pbch(fp->Lmax, Nid, n_hf, ssb_index);
    lastsymbol = 2;
    num_rbs = 20;

[....]
[....]
[....]

```

# Analysis:

- When we create logs within each `gold_generator` function, the idea is to see how the gold sequence is generated, specifically `x2`, and to see if there are any collisions.

- With the logs, we were able to prove at least that, for the same channel type for two UEs within the same cell, the UEs **use the same pseudorandom sequence for their pilots at that instant.**

- But the pilots themselves can be generated taking other values ​​into account. That's why we decided to include logs more focused on channel estimation.

- By further analyzing files like `nr_dl_channel_estimation`, we saw that many of the functions actually use the following method:

- They still use the `nr_gold` methods, and then insert a function like `nr_pdcch_dmrs_rx`, whose signature is as follows:
    ```c
      const uint32_t *gold = nr_gold_pdcch(ue->frame_parms.N_RB_DL, ue->frame_parms.symbols_per_slot, scrambling_id, slot, symbol);
      nr_pdcch_dmrs_rx(ue, slot, gold, (c16_t *)pilot, 2000, (nb_rb_coreset + dmrs_ref));
    ```
    
- In other words, it undergoes even more modulation: `nr_pbch_dmrs_rx(int ​​symbol, const unsigned int *nr_gold_pbch, c16_t *output, bool sidelink)`
- The pilot in our case will be the `output*`


### LOGS:

**For `nr_prs_channel_estimation()` we use:**

```c
      idx = (((gold_prs[(m << 1) >> 5]) >> ((m << 1) & 0x1f)) & 3);
      mod_prs[m] = nr_qpsk_mod_table[idx];
      LOG_I(PHY, "[PRS PILOT] slot %d, symb %d, pilot %d: QPSK idx %d → (%d, %d)", slot_prs, l, m, idx, mod_prs[m].r, mod_prs[m].i);
```

These are the sequence values ​​converted to QPSK.

**For `nr_pbch_dmrs_correlation` we use:**

```c

  for (int i = 0; i < 50; i++) {
    LOG_I(PHY, "[PBCH-DMRS PILOT] [%d] = (%d, %d)", i, pilot[i].r, pilot[i].i);
  }
```

**For `nr_pbch_channel_estimation` we use:**

```c
  for (int i = 0; i < 10; i++) {
    LOG_I(PHY, "[PBCH-DMRS PILOT - not corr] [%d] = (%d, %d)", i, pilot[i].r, pilot[i].i);
  }
```

**For `nr_pdcch_channel_estimation` we use:**

```c

  for (int i = 0; i < 50; i++) {
    LOG_I(PHY, "[PDCCH-DMRS pilot] [%d] = (%d, %d)", i, ((c16_t*)pilot)[i].r, ((c16_t*)pilot)[i].i);
  }
```

**For `nr_pdsch_channel_estimation` we use:**

```c
  for (int i = 0; i < 10; i++) {
    LOG_I(PHY, "[PDSCH-DMRS pilot] [%d] = (%d, %d)", i, pilot[i].r, pilot[i].i);
  }
```


Estes logs não funcionaram não sei porque… No entanto tentamos seguir as funções que são referenciadas de forma a tentar obter uma resposta.

Assim desta forma inserimos logs mais fundos nas funções:

| **Generate Function** | **Mod Function** |
| --- | --- |
| pbch | pbch_dmrs_rx |
| pbch_not_corr | pbch_dmrs_rx |
| pdcch | pdcch_dmrs_rx |
| pdsch | pdsch_dmrs_rx |

We inserted the following logs:

```c
 # PDSCH:
 printf("nr_pdsch_dmrs_rx dmrs config type %d port %d nb_pdsch_rb %d\n", config_type, p, nb_pdsch_rb);
 printf("wf[%d] = %d wt[%d]= %d\n", i & 1, wf[p - 1000][i & 1], lp, wt[p - 1000][lp]);
 printf("i %d pdsch mod_dmrs %d %d\n", i, output[i].r, output[i].i);
 
 # PBCH not corr:
- equal to PBCH
 
 # PDCCH:
 printf("i %d pdcch mod_dmrs %d %d\n", i, output[i].r, output[i].i);
 
 # PPBCH:
 printf("nr_gold_pbch[(m<<1)>>5] %x\n",nr_gold_pbch[(m<<1)>>5]);
 printf("m %d  output %d %d\n", m, output[m].r, output[m].i);
```

`PRS` does not appear, but in the logs it appears in the first messages:

```c
10121.807974 [PHY] I prs_config configuration NOT found..!! Skipped configuring UE for the PRS reception
```
Otherwise, we were able to do the analysis and even draw some graphs - because we were able to put simple logs - like the ones we saw above:

```c
Index,I Part,Q Part
0,-23170,-23170
1,23170,-23170
2,-23170,23170
3,-23170,-23170
4,23170,-23170
5,-23170,23170
6,23170,-23170
7,-23170,23170
8,23170,23170

[....]
```

- - These data are already quantized values, we tried to draw constellations with these by normalizing the data to 1, and we had the following results:

<img src="https://github.com/Surpr1se0/mosaic5G-docs/blob/main/img/constelation.png" alt="flexric" width="500">

- From these graphs, you can see that these signals are the internally calculated pilots, because they're so perfect. These pilots all look the same because they're not the received signals.

- These are calculated based on local parameters, as we've seen. And they're used to correlate with the received signal—that is, the received signals **will pass through the channel (noise, fading, etc.) and contain the modulated and attenuated pilots.**

- This way, we tried to increase the level of detail in the logs by trying to receive the pilots received at the antennas. We only achieved this when I finally had the idea of ​​switching to a channel sim—Rayleigh1.

```c
channelmod = {
  max_chan = 10;
  modellist = "modellist_rfsimu_1";
  modellist_rfsimu_1 = (
    { # DL, modify on UE side
      model_name     = "rfsimu_channel_enB0"
      type           = "AWGN";
      ploss_dB       = 20;
      noise_power_dB = -4;
      forgetfact     = 0;
      offset         = 0;
      ds_tdl         = 0;
    },
    { # UL, modify on gNB side
      model_name     = "rfsimu_channel_ue0"
      type           = "Rayleigh1";
      ploss_dB       = 0;
      noise_power_dB = -20;
      forgetfact     = 0;
      offset         = 0;
      ds_tdl         = 0;
    }
  );
};
```

# Sources:

[Wikipedia Gold Code](https://en.wikipedia.org/wiki/Gold_code)

[What are gold codes and how are they used](https://ham.stackexchange.com/questions/21599/what-exactly-are-gold-sequences-and-how-are-they-)

[ShareTechNote PHY Reference Signals](https://gitlab.eurecom.fr/oai/openairinterface5g/-/tree/develop/openair1?ref_type=heads)

[ShareTechNote Reference Signals 5G](https://www.sharetechnote.com/html/5G/5G_Phy_ReferenceSignal.html)

[ShareTechNote PHY Pseudorandom sequences](https://www.sharetechnote.com/html/5G/5G_Phy_PseudoRandomSequence.html)