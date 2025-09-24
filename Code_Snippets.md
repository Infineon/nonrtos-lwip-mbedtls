# Non-RTOS Code Snippets
In Non-RTOS mode, the netconn and socket APIs wont be available in lwIP. The application shall use the raw APIs for data transfer. Also application shall call `sys_check_timeouts` perioducally in the main loop. A typical startup of the program is as follows.

```
#include <stdio.h>
#include "cybsp.h"
#include "cy_utils.h"

#include "cy_network_mw_core.h"
#include "cy_eth_config.h"
#include "cy_nw_helper.h"
#include "cy_log.h"
#include "cy_result.h"

#include "cy_retarget_io.h"

#include "lwip/api.h"
#include "lwip/err.h"
#include "lwip/dns.h"
#include "lwip/timeouts.h"

#if (defined(eth_0_mux_0_RXD_POLLING_EN) && (eth_0_mux_0_RXD_POLLING_EN == 1u))
extern void ETH_LWIP_Poll(void);
#endif

/* MAC address*/
#define MAC_ADDR0                                (0x00U)
#define MAC_ADDR1                                (0x03U)
#define MAC_ADDR2                                (0x19U)
#define MAC_ADDR3                                (0x45U)
#define MAC_ADDR4                                (0x00U)
#define MAC_ADDR5                                (0x00U)

static void task_func(void)
{
    cy_rslt_t result = CY_RSLT_SUCCESS;
    cy_network_interface_context *iface_context;
    uint8_t mac_address[6];
    char ip_str[15];
    cy_nw_ip_address_t ipv4_addr, ipv4_gateway_addr;
    cy_nw_ip_mac_t nw_mac_addr;
    ETH_LWIP_t eth_lwip;

    mac_address[0] = (uint8_t)MAC_ADDR0;
    mac_address[1] = (uint8_t)MAC_ADDR1;
    mac_address[2] = (uint8_t)MAC_ADDR2;
    mac_address[3] = (uint8_t)MAC_ADDR3;
    mac_address[4] = (uint8_t)MAC_ADDR4;
    mac_address[5] = (uint8_t)MAC_ADDR5;

    printf(" SystemCoreClock:%ld\n", (long)SystemCoreClock);

    /* NW init */
    result = cy_network_init();
    if(result != CY_RSLT_SUCCESS)
    {
        printf("cy_network_init failed result:0x%lX\n", result );
        goto exit ;
    }

    printf("\ncy_network_init success\n");
    eth_lwip.initialized = false;

    /* Add Ethernet interface */
    result = cy_network_add_nw_interface( CY_NETWORK_ETH_INTERFACE, 0, &eth_lwip, &(mac_address[0]), NULL, &iface_context );
    if(result != CY_RSLT_SUCCESS)
    {
        printf("failed to add the network interface result:0x%lX\n", result );
        goto exit ;
    }

    /* NW IP UP */
    result = cy_network_ip_up( iface_context );
    if(result != CY_RSLT_SUCCESS)
    {
        printf("cy_network_ip_up failed result:0x%lX\n", result );
        goto exit ;
    }

    /* Device IP */
    result = cy_network_get_ip_address( iface_context, &ipv4_addr );
    if( result == CY_RSLT_SUCCESS )
    {
        cy_nw_ntoa( &ipv4_addr, ip_str );
        printf( "IP Address %s assigned \n", ip_str );
    }

    /* Gateway IP */
    result = cy_network_get_gateway_ip_address( iface_context, &ipv4_gateway_addr );
    if( result != CY_RSLT_SUCCESS )
    {
        printf( "Failed to get the gateway address\n" );
        goto exit;
    }
    else
    {
        cy_nw_ntoa( &ipv4_gateway_addr, ip_str );
        printf( "GW Address %s assigned \n", ip_str );
    }

    /* Gateway MAC */
    result = cy_network_get_gateway_mac_address( iface_context, &nw_mac_addr );
    if( result != CY_RSLT_SUCCESS )
    {
        printf( "Failed to get the gateway mac address address\n" );
        goto exit;
    }
    printf("MAC address of gateway = %02X:%02X:%02X:%02X:%02X:%02X\n",
            nw_mac_addr.mac[0], nw_mac_addr.mac[1], nw_mac_addr.mac[2],
            nw_mac_addr.mac[3], nw_mac_addr.mac[4], nw_mac_addr.mac[5]);

    /* Place holder for starting Application functions */

exit:
    while(1)
    {
#if (defined(eth_0_mux_0_RXD_POLLING_EN) && (eth_0_mux_0_RXD_POLLING_EN == 1u))
        ETH_LWIP_Poll();
#endif
        sys_check_timeouts();
    }
}

/* Main function */
int main(void)
{
    cy_rslt_t result;

    /* Initialize the device and board peripherals */
    result = cybsp_init();
    if (result != CY_RSLT_SUCCESS)
    {
        CY_ASSERT(0);
    }

    cy_log_init( CY_LOG_DEBUG, NULL, NULL );

    /* Initialize printf retarget */
    cy_retarget_io_init(CYBSP_DEBUG_UART_HW);

    __enable_irq();

    printf("Non-RTOS Test Application Entry!\n");

    /* Run the task */
    task_func();

    return 0;
}
```

### Ping Test Code
The below code demonstrates periodic ping to the configured IP address (PING_IP_ADDRESS) on the user defined interval (PING_DELAY). `ping_start_periodic` shall be called after initializations to trigger the ping.

```
#include "lwip/mem.h"
#include "lwip/raw.h"
#include "lwip/icmp.h"
#include "lwip/netif.h"
#include "lwip/sys.h"
#include "lwip/timeouts.h"
#include "lwip/inet_chksum.h"
#include "lwip/prot/ip4.h"


/** ping receive timeout - in milliseconds */
#ifndef PING_RCV_TIMEO
#define PING_RCV_TIMEO 1000
#endif

/** ping delay - in milliseconds */
#ifndef PING_DELAY
#define PING_DELAY     10000
#endif

/** ping identifier - must fit on a u16_t */
#ifndef PING_ID
#define PING_ID        0xAFAF
#endif

/** ping additional data size to include in the packet */
#ifndef PING_DATA_SIZE
#define PING_DATA_SIZE 32
#endif

/** ping result action - no default action */
#ifndef PING_RESULT
#define PING_RESULT(ping_ok)
#endif

/** Enable Debugging */
#define PING_DEBUGF    printf

static ip_addr_t* ping_target = NULL;
static u16_t ping_seq_num;
static struct raw_pcb *ping_pcb;

static u32_t ping_time;

#define MAKE_IPV4_ADDRESS(a, b, c, d) ((((uint32_t) d) << 24) | (((uint32_t) c) << 16) | (((uint32_t) b) << 8) |((uint32_t) a))

/* IP Address of the target */
#define PING_IP_ADDRESS MAKE_IPV4_ADDRESS(192,168,1,1)

static void print_ipaddr(const ip_addr_t *addr)
{
  uint8_t bytes[4];
  uint32_t ip = addr->addr;

  bytes[0] = ip & 0xFF;
  bytes[1] = (ip >> 8) & 0xFF;
  bytes[2] = (ip >> 16) & 0xFF;
  bytes[3] = (ip >> 24) & 0xFF;

  PING_DEBUGF("%u.%u.%u.%u ", bytes[0], bytes[1], bytes[2], bytes[3]);
}

static void
ping_req_timeout(void *arg)
{
  PING_DEBUGF("ping: timeout \n");
}

static u8_t
ping_recv(void *arg, struct raw_pcb *pcb, struct pbuf *p, const ip_addr_t *addr)
{
  struct icmp_echo_hdr *iecho;
  LWIP_UNUSED_ARG(arg);
  LWIP_UNUSED_ARG(pcb);
  LWIP_UNUSED_ARG(addr);
  LWIP_ASSERT("p != NULL", p != NULL);

  if ((p->tot_len >= (PBUF_IP_HLEN + sizeof(struct icmp_echo_hdr))) &&
      pbuf_remove_header(p, PBUF_IP_HLEN) == 0) {
    iecho = (struct icmp_echo_hdr *)p->payload;

    if ((iecho->id == PING_ID) && (iecho->seqno == lwip_htons(ping_seq_num))) {
      PING_DEBUGF("ping: reply from ");
      print_ipaddr(addr);
      PING_DEBUGF("bytes=%d ", p->tot_len - sizeof(struct icmp_echo_hdr));
      PING_DEBUGF(" time=%"U32_F"ms\n", (sys_now()-ping_time));

      /* do some ping result processing */
      PING_RESULT(1);
      pbuf_free(p);
      sys_untimeout(ping_req_timeout, NULL);
      return 1; /* eat the packet */
    }
    /* not eaten, restore original packet */
    pbuf_add_header(p, PBUF_IP_HLEN);
  }

  return 0; /* don't eat the packet */
}

/** Prepare a echo ICMP request */
static void
ping_prepare_echo( struct icmp_echo_hdr *iecho, u16_t len)
{
  size_t i;
  size_t data_len = len - sizeof(struct icmp_echo_hdr);

  ICMPH_TYPE_SET(iecho, ICMP_ECHO);
  ICMPH_CODE_SET(iecho, 0);
  iecho->chksum = 0;
  iecho->id     = PING_ID;
  iecho->seqno  = lwip_htons(++ping_seq_num);

  /* fill the additional data buffer with some data */
  for(i = 0; i < data_len; i++) {
    ((char*)iecho)[sizeof(struct icmp_echo_hdr) + i] = (char)i;
  }
#ifndef COMPONENT_CAT3
  iecho->chksum = inet_chksum(iecho, len);
#endif
}


static void
ping_send(struct raw_pcb *raw, const ip_addr_t *addr)
{
  struct pbuf *p;
  struct icmp_echo_hdr *iecho;
  size_t ping_size = sizeof(struct icmp_echo_hdr) + PING_DATA_SIZE;

  PING_DEBUGF("ping: send ");
  print_ipaddr(addr);
  PING_DEBUGF("\n");
  LWIP_ASSERT("ping_size <= 0xffff", ping_size <= 0xffff);

  p = pbuf_alloc(PBUF_IP, (u16_t)ping_size, PBUF_RAM);
  if (!p) {
    return;
  }
  if ((p->len == p->tot_len) && (p->next == NULL)) {
    iecho = (struct icmp_echo_hdr *)p->payload;

    ping_prepare_echo(iecho, (u16_t)ping_size);

    raw_sendto(raw, p, addr);

    ping_time = sys_now();
  }
  pbuf_free(p);
}

static void
ping_timeout(void *arg)
{
  struct raw_pcb *pcb = (struct raw_pcb*)arg;

  LWIP_ASSERT("ping_timeout: no pcb given!", pcb != NULL);

  ping_send(pcb, ping_target);

  /* Start recv timeout */
  sys_timeout(PING_RCV_TIMEO, ping_req_timeout, NULL);

  sys_timeout(PING_DELAY, ping_timeout, pcb);
}

static void
ping_raw_init(void)
{
  ping_pcb = raw_new(IP_PROTO_ICMP);
  LWIP_ASSERT("ping_pcb != NULL", ping_pcb != NULL);

  raw_recv(ping_pcb, ping_recv, NULL);
  raw_bind(ping_pcb, IP_ADDR_ANY);
  sys_timeout(PING_DELAY, ping_timeout, ping_pcb);
}

/* Initialize Ping */
void ping_init(ip_addr_t* ping_addr)
{
  if(ping_target == NULL)
  {
    ping_target = (ip_addr_t*)mem_malloc((mem_size_t)sizeof(ip_addr_t));
    memcpy(ping_target, ping_addr, sizeof(ip_addr_t));
    ping_raw_init();
  }
}

/* Send a ping request */
void ping_send_now(void)
{
  LWIP_ASSERT("ping_pcb != NULL", ping_pcb != NULL);
  ping_send(ping_pcb, ping_target);
}


void
ping_start_periodic(void)
{
    ip_addr_t ipaddr;

    ipaddr.addr = PING_IP_ADDRESS;
    ping_init(&ipaddr);
    ping_send_now();
}
```


### HTTP Client Test Code
The below code demonstrates the connection to httpbin.org via HTTP protocol. The `http_test` function shall be called after all the initialization is done.

**Note** : This test code requires a couple of files from lwIP application folder. Please copy `http_client.c`, `httpd_structs.h` and `httpd.c` from `lwip\src\apps\http` to the application folder.

```
#include <stdio.h>

#include "lwip/apps/http_client.h"

static uint8_t header_buffer[768];
static uint8_t data_buffer[1024];
static httpc_connection_t settings;

extern err_t httpc_get_file_dns(const char* server_name, u16_t port, const char* uri, const httpc_connection_t *settings,
                   altcp_recv_fn recv_fn, void* callback_arg, httpc_state_t **connection);

/* HTTP Recv Callback */
err_t rec_cb(void *arg, struct altcp_pcb *conn, struct pbuf *p, err_t err)
{
    printf("rec_cb: len %d\n", p->len);
    struct pbuf *q;
    uint32_t framelen=0;

    memset(data_buffer, 0, sizeof(data_buffer));

    for(q = p; q != NULL; q = q->next)
    {
        MEMCPY(data_buffer, p->payload, p->len);
        framelen += (uint32_t)p->len;
    }

    printf("%s\n", data_buffer);
    printf("\n");

    free(p);
    return ERR_OK;
}

/* HTTP headers done Callback */
err_t headers_done_cb(httpc_state_t *connection, void *arg, struct pbuf *hdr, u16_t hdr_len, u32_t content_len)
{
    printf("header length: %d\n", hdr_len);
    printf("content length in the header: %ld\n", content_len);

    struct pbuf *q;
    uint32_t framelen=0;

    memset(header_buffer, 0, sizeof(header_buffer));

    for(q = hdr; q != NULL; q = q->next)
    {
        MEMCPY(header_buffer, q->payload, q->len);
        framelen += (uint32_t)q->len;
    }

    printf("%s\n", header_buffer);
    printf("\n");

    return ERR_OK;
}

/* Result Callback */
void result_cb(void *arg, httpc_result_t httpc_result, u32_t rx_content_len, u32_t srv_res, err_t err)
{
    if(err == ERR_OK)
    {
        printf("download finished\n");
    }
    else
    {
        printf("Download err - %d\n", err);
    }
}

void http_test(void)
{
    const char* server_name = "www.httpbin.org";
    u16_t port = 80;
    const char* uri = "/anything";
    httpc_state_t *httpc_state = NULL;
    err_t err;
    #if LWIP_ALTCP
    settings.altcp_allocator = NULL;
    #endif
    settings.use_proxy = 0;
    settings.headers_done_fn = headers_done_cb;
    settings.result_fn = result_cb;

    /* http get with DNS resolution */
    err = httpc_get_file_dns(server_name, port, uri, &settings, rec_cb, NULL, &httpc_state);
    if ( err != ERR_OK )
    {
        printf("httpc_get_file_dns failed (%d)", err);
    }
    else
    {
        printf("httpc_get_file_dns passed (%d)\n", err);
    }
}
```


### HTTPS Client Test Code
The below code demonstrates the connection to httpbin.org via HTTPS protocol. The `https_test` function shall be called from the application once all the initialization is complete.

#### prerequisite ####
1. Copy `altcp_tls` folder from `lwip/src/apps/altcp_tls` to application folder.
2. Add the following defines to lwipopts.h

```
#define LWIP_ALTCP                    (1)
#define LWIP_ALTCP_TLS                (1)
#define LWIP_ALTCP_TLS_MBEDTLS        (1)

```

3. Apply the below changes to `altcp_tls_mbedtls.c`

```
--- a/altcp_tls/altcp_tls_mbedtls.c
+++ b/altcp_tls/altcp_tls_mbedtls.c
@@ -69,10 +69,8 @@
 /* @todo: which includes are really needed? */
 #include "mbedtls/entropy.h"
 #include "mbedtls/ctr_drbg.h"
-#include "mbedtls/certs.h"
 #include "mbedtls/x509.h"
 #include "mbedtls/ssl.h"
-#include "mbedtls/net.h"
 #include "mbedtls/error.h"
 #include "mbedtls/debug.h"
 #include "mbedtls/platform.h"
@@ -80,7 +78,6 @@
 #include "mbedtls/ssl_cache.h"
 #include "mbedtls/net_sockets.h"

-#include "mbedtls/ssl_internal.h" /* to call mbedtls_flush_output after ERR_MEM */

 #include <string.h>

@@ -116,6 +113,23 @@ static err_t altcp_mbedtls_handle_rx_appldata(struct altcp_pcb *conn, altcp_mbed
 static int altcp_mbedtls_bio_send(void *ctx, const unsigned char *dataptr, size_t size);


+int mbedtls_test_rand(void *rng_state,
+                              unsigned char *output,
+                              size_t len)
+{
+    size_t i;
+
+    if (rng_state != NULL) {
+        rng_state  = NULL;
+    }
+
+    for (i = 0; i < len; ++i) {
+        output[i] = rand();
+    }
+
+    return 0;
+}
+
 /* callback functions from inner/lower connection: */

 /** Accept callback from lower connection (i.e. TCP)
@@ -678,7 +692,7 @@ altcp_tls_create_config(int is_server, int have_cert, int have_pkey, int have_ca
   struct altcp_tls_config *conf;
   mbedtls_x509_crt *mem;

-  if (TCP_WND < MBEDTLS_SSL_MAX_CONTENT_LEN) {
+  if (TCP_WND < MBEDTLS_SSL_IN_CONTENT_LEN) {
     LWIP_DEBUGF(ALTCP_MBEDTLS_DEBUG|LWIP_DBG_LEVEL_SERIOUS,
       ("altcp_tls: TCP_WND is smaller than the RX decryption buffer, connection RX might stall!\n"));
   }
@@ -779,7 +793,7 @@ altcp_tls_create_config_server_privkey_cert(const u8_t *privkey, size_t privkey_
     return NULL;
   }

-  ret = mbedtls_pk_parse_key(pkey, (const unsigned char *) privkey, privkey_len, privkey_pass, privkey_pass_len);
+  ret = mbedtls_pk_parse_key(pkey, (const unsigned char *) privkey, privkey_len, privkey_pass, privkey_pass_len, mbedtls_test_rand, NULL);
   if (ret != 0) {
     LWIP_DEBUGF(ALTCP_MBEDTLS_DEBUG, ("mbedtls_pk_parse_public_key failed: %d\n", ret));
     mbedtls_x509_crt_free(srvcert);
@@ -859,7 +873,7 @@ altcp_tls_create_config_client_2wayauth(const u8_t *ca, size_t ca_len, const u8_
   }

   mbedtls_pk_init(conf->pkey);
-  ret = mbedtls_pk_parse_key(conf->pkey, privkey, privkey_len, privkey_pass, privkey_pass_len);
+  ret = mbedtls_pk_parse_key(conf->pkey, privkey, privkey_len, privkey_pass, privkey_pass_len, mbedtls_test_rand, NULL);
   if (ret != 0) {
     LWIP_DEBUGF(ALTCP_MBEDTLS_DEBUG, ("mbedtls_pk_parse_key failed: %d 0x%x", ret, -1*ret));
     altcp_mbedtls_free_config(conf);
@@ -1053,9 +1067,9 @@ altcp_mbedtls_write(struct altcp_pcb *conn, const void *dataptr, u16_t len, u8_t
   /* HACK: if thre is something left to send, try to flush it and only
      allow sending more if this succeeded (this is a hack because neither
      returning 0 nor MBEDTLS_ERR_SSL_WANT_WRITE worked for me) */
-  if (state->ssl_context.out_left) {
+  if (state->ssl_context.MBEDTLS_PRIVATE(out_left)) {
     mbedtls_ssl_flush_output(&state->ssl_context);
-    if (state->ssl_context.out_left) {
+    if (state->ssl_context.MBEDTLS_PRIVATE(out_left)) {
       return ERR_MEM;
     }

```

#### Code ####

```
/*
 * @file : https_test.h
 *
 */

#ifndef HTTPS_TEST_H_
#define HTTPS_TEST_H_


/* HTTPS server name */
#define HTTPS_HOSTNAME                          "httpbin.org"

/* CA Certificate */
#define HTTPS_CA_ROOT_CERT \
"-----BEGIN CERTIFICATE-----\n" \
"MIIDQTCCAimgAwIBAgITBmyfz5m/jAo54vB4ikPmljZbyjANBgkqhkiG9w0BAQsF\n" \
"ADA5MQswCQYDVQQGEwJVUzEPMA0GA1UEChMGQW1hem9uMRkwFwYDVQQDExBBbWF6\n" \
"b24gUm9vdCBDQSAxMB4XDTE1MDUyNjAwMDAwMFoXDTM4MDExNzAwMDAwMFowOTEL\n" \
"MAkGA1UEBhMCVVMxDzANBgNVBAoTBkFtYXpvbjEZMBcGA1UEAxMQQW1hem9uIFJv\n" \
"b3QgQ0EgMTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBALJ4gHHKeNXj\n" \
"ca9HgFB0fW7Y14h29Jlo91ghYPl0hAEvrAIthtOgQ3pOsqTQNroBvo3bSMgHFzZM\n" \
"9O6II8c+6zf1tRn4SWiw3te5djgdYZ6k/oI2peVKVuRF4fn9tBb6dNqcmzU5L/qw\n" \
"IFAGbHrQgLKm+a/sRxmPUDgH3KKHOVj4utWp+UhnMJbulHheb4mjUcAwhmahRWa6\n" \
"VOujw5H5SNz/0egwLX0tdHA114gk957EWW67c4cX8jJGKLhD+rcdqsq08p8kDi1L\n" \
"93FcXmn/6pUCyziKrlA4b9v7LWIbxcceVOF34GfID5yHI9Y/QCB/IIDEgEw+OyQm\n" \
"jgSubJrIqg0CAwEAAaNCMEAwDwYDVR0TAQH/BAUwAwEB/zAOBgNVHQ8BAf8EBAMC\n" \
"AYYwHQYDVR0OBBYEFIQYzIU07LwMlJQuCFmcx7IQTgoIMA0GCSqGSIb3DQEBCwUA\n" \
"A4IBAQCY8jdaQZChGsV2USggNiMOruYou6r4lK5IpDB/G/wkjUu0yKGX9rbxenDI\n" \
"U5PMCCjjmCXPI6T53iHTfIUJrU6adTrCC2qJeHZERxhlbI1Bjjt/msv0tadQ1wUs\n" \
"N+gDS63pYaACbvXy8MWy7Vu33PqUXHeeE6V/Uq2V8viTO96LXFvKWlJbYK8U90vv\n" \
"o/ufQJVtMVT8QtPHRh8jrdkPSHCa2XV4cdFyQzR1bldZwgJcJmApzyMZFo6IQ6XU\n" \
"5MsI+yMRQ+hDKXJioaldXgjUkK642M4UwtBV8ob2xJNDd2ZhwLnoQdeXeGADbkpy\n" \
"rqXRfboQnoZsG4q5WTP468SQvvG5\n" \
"-----END CERTIFICATE-----\n"

/* idle connection polling interval */
#define HTTPS_ALTCP_IDLE_POLL_INTERVAL          (2)

/* HTTP request */
#define HTTPS_REQUEST                       \
    "GET /anything HTTP/1.1\r\n"            \
    "Accept: */*\r\n"                       \
    "Host: " HTTPS_HOSTNAME "\r\n"          \
    "Connection: Close\r\n"                 \
    "\r\n"

/* lwIP errors */
typedef err_t lwip_err_t;

/* MbedTLS errors */
typedef int mbedtls_err_t;

typedef struct altcp_callback_arg
{
    /* connection configuration */
    struct altcp_tls_config* config;

    /* connection state */
    bool connected;

    /* connection failure */
    bool conn_fail;
}altcp_callback_arg_t;

/* Test Function */
void https_test(void);

#endif /* HTTPS_TEST_H_ */


/*
 * @file : https_test.c
 *
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

/* lwIP includes */
#include "lwip/dns.h"
#include "lwip/altcp.h"
#include "lwip/altcp_tls.h"
#include "altcp_tls_mbedtls_structs.h"
#include "lwip/prot/iana.h"
#include "lwip/timeouts.h"

/* MbedTLS includes */
#include "mbedtls/ssl.h"
#include "mbedtls/check_config.h"

#include "psa/crypto.h"

#include "https_test.h"

#define CY_GET_LEN(array) (sizeof array) / (sizeof array[0])

/* Globals */
static u8_t ca_cert[] = HTTPS_CA_ROOT_CERT;
static bool dns_res_done = false;

#if (defined(eth_0_mux_0_RXD_POLLING_EN) && (eth_0_mux_0_RXD_POLLING_EN == 1u))
extern void ETH_LWIP_Poll(void);
#endif

int mbedtls_hardware_poll(void *data, unsigned char *output, size_t len, size_t *olen)
{
    uint8_t rand_num[200] = {0};
    int lower = 1, upper = 100;

#if !defined(__ARMCC_VERSION)
    srand(time(NULL));
#endif
    for (int i = 0; i < len; i++)
    {
        rand_num[i] = (uint8_t)(rand() % (upper - lower + 1)) + lower;
    }
    memcpy(output, rand_num, len);

    *olen = len;
    return 0;
}

static void wait_for_input(void)
{
#if (defined(eth_0_mux_0_RXD_POLLING_EN) && (eth_0_mux_0_RXD_POLLING_EN == 1u))
    ETH_LWIP_Poll();
#endif
    sys_check_timeouts();
}

// connection establishment callback
static lwip_err_t altcp_connect_callback(void *arg, struct altcp_pcb *pcb, lwip_err_t err)
{
    altcp_callback_arg_t *clbk = (altcp_callback_arg_t *)arg;
    clbk->connected = true;
    return ERR_OK;
}

// Free protocol control block
static void altcp_free_pcb(struct altcp_pcb *pcb)
{
    altcp_close(pcb);
}

// Free connection configuration
static void altcp_free_config(struct altcp_tls_config *config)
{
    altcp_tls_free_config(config);
}

// Free connection callback argument
static void altcp_free_arg(altcp_callback_arg_t *arg)
{
    if (arg)
    {
        free(arg);
    }
}

// DNS response callback
static void gethostbyname_callback(const char *name, const ip_addr_t *resolved, void *ipaddr)
{
    if (resolved)
    {
        *((ip_addr_t *)ipaddr) = *resolved;
    }
    else
    {
        ((ip_addr_t *)ipaddr)->addr = IPADDR_NONE;
    }
    dns_res_done = true;
}

static bool resolve_hostname(ip_addr_t *ipaddr)
{
    lwip_err_t lwip_err;

    ipaddr->addr = IPADDR_ANY;

    lwip_err = dns_gethostbyname(HTTPS_HOSTNAME, ipaddr, gethostbyname_callback, ipaddr);
    if (lwip_err == ERR_INPROGRESS)
    {
        while (ipaddr->addr == IPADDR_ANY && dns_res_done == false)
        {
            wait_for_input();
        }

        lwip_err = ERR_OK;
        /* see if dns is resolved */
        if (ipaddr->addr == IPADDR_ANY)
        {
            lwip_err = ERR_IF;
        }
    }
    return !((bool)lwip_err);
}

// Send HTTPS request
static bool send_request(struct altcp_pcb *pcb)
{
    lwip_err_t lwip_err;
    const char request[] = HTTPS_REQUEST;

    lwip_err = altcp_write(pcb, request, CY_GET_LEN(request) - 1, 0);
    if (lwip_err == ERR_OK)
    {
        /* Output send buffer */
        lwip_err = altcp_output(pcb);
    }
    return !((bool)lwip_err);
}

// connection error callback
static void altcp_err_callback(void *arg, lwip_err_t err)
{
    altcp_callback_arg_t *clbk_arg = (altcp_callback_arg_t *)arg;

    printf("Connection error [err = %d]\n", err);

    clbk_arg->conn_fail = true;

    if (clbk_arg->config)
    {
        altcp_free_config(clbk_arg->config);
    }
    altcp_free_arg(clbk_arg);
}

// connection idle callback
static lwip_err_t altcp_poll_callback(void *arg, struct altcp_pcb *pcb)
{
    return ERR_OK;
}

// data acknowledgement callback
lwip_err_t altcp_sent_callback(void *arg, struct altcp_pcb *pcb, u16_t len)
{
    return ERR_OK;
}

// data recv callback
static lwip_err_t altcp_recv_callback(void *arg, struct altcp_pcb *pcb, struct pbuf *buf, lwip_err_t err)
{
    struct pbuf *head = buf;

    switch (err)
    {
    case ERR_OK:
    {
        if (buf)
        {
            uint16_t i;
            while (buf->len != buf->tot_len)
            {
                for (i = 0; i < buf->len; i++)
                {
                    printf("%c", (((char *)buf->payload)[i]));
                }
                buf = buf->next;
            }
            for (i = 0; i < buf->len; i++)
            {
                printf("%c", (((char *)buf->payload)[i]));
            }
            altcp_recved(pcb, head->tot_len);
        }
        else
        {
            printf("Response completed\r\n");
        }
        break;
    }
    case ERR_ABRT:
    {
        err = ERR_OK;
        break;
    }
    }
    if (head)
    {
        pbuf_free(head);
    }
    return err;
}

// Establish connection with server
static bool connect_to_host(ip_addr_t *ipaddr, struct altcp_pcb **pcb)
{
    struct altcp_tls_config *config = altcp_tls_create_config_client(ca_cert, CY_GET_LEN(ca_cert));

    if (!config)
    {
        return false;
    }

    // Instantiate connection PCB
    *pcb = altcp_tls_new(config, IPADDR_TYPE_V4);
    if (!(*pcb))
    {
        altcp_free_config(config);
        return false;
    }

    mbedtls_ssl_context *context = &((altcp_mbedtls_state_t *)((*pcb)->state))->ssl_context;

    mbedtls_err_t mbedtls_err = mbedtls_ssl_set_hostname(context, HTTPS_HOSTNAME);
    if (mbedtls_err)
    {
        altcp_free_pcb(*pcb);
        altcp_free_config(config);
        return false;
    }

    // Configure common argument for connection callbacks
    struct altcp_callback_arg *arg = malloc(sizeof(*arg));
    if (!arg)
    {
        altcp_free_pcb(*pcb);
        altcp_free_config(config);
        return false;
    }
    arg->config = config;
    arg->connected = false;
    arg->conn_fail = false;
    altcp_arg(*pcb, (void *)arg);

    /* Configure connection fatal error callback */
    altcp_err(*pcb, altcp_err_callback);

    /* Configure idle connection callback (and interval) */
    altcp_poll(*pcb, altcp_poll_callback, HTTPS_ALTCP_IDLE_POLL_INTERVAL);

    /* Configure data acknowledge callback */
    altcp_sent(*pcb, altcp_sent_callback);

    /* Configure data reception callback */
    altcp_recv(*pcb, altcp_recv_callback);

    /* Send connection request */
    lwip_err_t lwip_err = altcp_connect(*pcb, ipaddr, LWIP_IANA_PORT_HTTPS, altcp_connect_callback);
    if (lwip_err == ERR_OK)
    {
        /* Wait for connection completion */
        while (!(arg->connected) && !(arg->conn_fail))
        {
            wait_for_input();
        }
        if (arg->conn_fail)
        {
            lwip_err = ERR_CONN;
        }
    }
    else
    {
        altcp_free_pcb(*pcb);
        altcp_free_config(config);
        altcp_free_arg(arg);
    }
    return !((bool)lwip_err);
}

/* HTTPS test function */
void https_test(void)
{
    ip_addr_t ipaddr;
    char *s_ipaddr;
    altcp_callback_arg_t *clbk_arg;
    psa_status_t psa_status;

    /* DNS resolution */
    if (!resolve_hostname(&ipaddr))
    {
        printf("Failed to resolve %s\n", HTTPS_HOSTNAME);
        return;
    }
    s_ipaddr = ipaddr_ntoa(&ipaddr);
    printf("Resolved %s (%s)\n", HTTPS_HOSTNAME, s_ipaddr);

    psa_status = psa_crypto_init();
    if (PSA_SUCCESS != psa_status)
    {
        printf("Failed to initialize PSA crypto : %ld \r\n", psa_status);
        return;
    }

    /* Establish connection with server */
    struct altcp_pcb *pcb = NULL;
    printf("Connecting to https://%s:%d\n", s_ipaddr, LWIP_IANA_PORT_HTTPS);
    if (!connect_to_host(&ipaddr, &pcb))
    {
        printf("Failed to connect to https://%s:%d\n", s_ipaddr, LWIP_IANA_PORT_HTTPS);
        return;
    }
    printf("Connected to https://%s:%d\n", s_ipaddr, LWIP_IANA_PORT_HTTPS);

    /* Send HTTP request to server */
    printf("Sending request\n");
    if (!send_request(pcb))
    {
        printf("Failed to send request\n");

        clbk_arg = (altcp_callback_arg_t *)(pcb->arg);
        altcp_free_config(clbk_arg->config);
        altcp_free_arg(clbk_arg);
        altcp_free_pcb(pcb);
        return;
    }
    printf("HTTPS Request sent\n");
}

```