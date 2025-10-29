# Asynchronous NoC extension
**Extension State: Under Development; EXTREMELY WIP**

TODO:
- interrupt mask for int when done

Depends on: NoC

## Why?
This extension allows programs to receive NoC data whilst doing something else,
making NoC way more useful.

Implementing this extension **without** having core-local memory is __stupid__

Implementing this extension in an out-of-order CPU isn't smart either, because you could have multiple execution ports that perform normal NoC send or receive instructions.

## Specification
This can only be used in **priviliged mode**, for simplicity reasons.

There are `NUM_NOC_ASYNC` async NoC controllers, which is at least one.

You don't need to NFLSH after async noc send

If there are multiple async NoC controllers, this can also be used to multi-cast packets faster.

All the memory reads or writes that the async NoC controller performs (as commanded by the system level application), will be performed in system level.

## Added Contol Registers
| Name                 | Description   |
| -------------------- | ------------- |
| `NUM_NOC_ASYNC`      | amount of async NoC controllers |
| `NOC_ASYNC_CFG_BASE` | pointer to the memory mapped async NoC controllers |

## Memory Mapped
A contiguos array of `noc_ctl_t` are located at `NOC_ASYNC_CFG_BASE`, with the length of `NUM_NOC_ASYNC`.
Reading or writing to/from that memory area in user mode triggeres a GP fault.
Reading from a field of a NoC controller is undefined, except for `mult`, which returns how many packets remain for sending to the async NoC controller.
Writing to the `mult` field will start the async NoC controller.
Writing to the other fields configures the async NoC controller.

When writing to the buf field, you have to write the correct variant of the union, based on your current address mode. The hardware sign extends that to the physical address size.

```c
typedef struct {
  uint8_t  /* bool */ send;
  union {
    uint16_t a16;
    uint32_t a32;
    uint64_t a64;
  } buf; /* char* */
  volatile uint16_t mult;
  uint8_t  nocport;
  uint16_t target;
} __attribute__((packed)) noc_ctl_t;
```

## Example
```c
#if sizeof(void*) == 2
# define ADDR_SIZE 16
#elif sizeof(void*) == 4
# define ADDR_SIZE 32
#elif sizeof(void*) == 8
# define ADDR_SIZE 64
#endif

void noc_recv(char * data, uint8_t port); // NRCV
uint8_t noc_available_mask(); // NAVL
void noc_send(uint16_t target_id, uint8_t target_port, char * data); // NSND
void noc_flush(uint16_t target_id, uint8_t target_port); // NFLSH
void noc_async_cfg(uint8_t ctl, bool send, char* buf, size_t mult, uint8_t nocport, uint16_t target) {
  noc_ctl_t* p = &((noc_ctl_t*)(void*)getcr(NOC_ASYNC_CFG_BASE))[ctl];
  p->send = send;
  p->buf.a##ADDR_SIZE = buf;
  p->nocport = nocport;
  p->target = target;
  p->mult = mult; // set this last!
}
size_t noc_async_rem(uint8_t ctl) {
  noc_ctl_t* p = &((noc_ctl_t*)(void*)getcr(NOC_ASYNC_CFG_BASE))[ctl];
  return p->mult;
}

// not optimal at all!
uint8_t noc_sendn(uint16_t target_id, uint8_t target_port, char * data, size_t data_len) {
  if (data_len % getcr(NOC_LEN) != 0) {
    data_len /= getcr(NOC_LEN);
    data_len ++;
    data_len *= getcr(NOC_LEN);
  }

  // should also consider using other ports
  if (has_ext(ASYNC_NOC) && noc_async_rem(0) == 0) {
    noc_async_cfg(0, true, data, data_len / getcr(NOC_LEN), target_port, target_id);
    return 0;
  }
  else {
    see NoC example code
  }
}

void example() {
  char buf[] = "A lot of dataaaaa";
  uint8_t ctl = noc_sendn(123, 1, buf, sizeof(buf));
  if (ctl != UINT8_MAX) {
    while (noc_async_rem(ctl) > 0) ;
  }
}
```
