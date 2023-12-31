.. _mozilla_projects_nss_nss_sample_code_enc_dec_mac_using_key_wrap_certreq_pkcs10_csr:

Enc Dec MAC Using Key Wrap CertReq PKCS10 CSR
=============================================

.. _nss_sample_code_6_encryptiondecryption_and_mac_and_output_public_as_a_pkcs_11_csr.:

`NSS Sample Code 6: Encryption/Decryption and MAC and output Public as a PKCS 11 CSR. <#nss_sample_code_6_encryptiondecryption_and_mac_and_output_public_as_a_pkcs_11_csr.>`__
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

.. container::

   Generates encryption/mac keys and outputs public key as pkcs11 certificate signing request

   .. code:: brush:

       /* This Source Code Form is subject to the terms of the Mozilla Public
        * License, v. 2.0. If a copy of the MPL was not distributed with this
        * file, You can obtain one at https://mozilla.org/MPL/2.0/. */

      /* NSPR Headers */
       #include <prthread.h>
       #include <plgetopt.h>
       #include <prerror.h>
       #include <prinit.h>
       #include <prlog.h>
       #include <prtypes.h>
       #include <plstr.h>

      /* NSS headers */
       #include <keyhi.h>
       #include <pk11priv.h>

      /* our samples utilities */
       #include "util.h"

      /* Constants */
       #define BLOCKSIZE             32
       #define MODBLOCKSIZE          128
       #define DEFAULT_KEY_BITS      1024

      /* Header file Constants */
       #define ENCKEY_HEADER         "-----BEGIN WRAPPED ENCKEY-----"
       #define ENCKEY_TRAILER        "-----END WRAPPED ENCKEY-----"
       #define MACKEY_HEADER         "-----BEGIN WRAPPED MACKEY-----"
       #define MACKEY_TRAILER        "-----END WRAPPED MACKEY-----"
       #define IV_HEADER             "-----BEGIN IV-----"
       #define IV_TRAILER            "-----END IV-----"
       #define MAC_HEADER            "-----BEGIN MAC-----"
       #define MAC_TRAILER           "-----END MAC-----"
       #define PAD_HEADER            "-----BEGIN PAD-----"
       #define PAD_TRAILER           "-----END PAD-----"
       #define LAB_HEADER            "-----BEGIN KEY LABEL-----"
       #define LAB_TRAILER           "-----END KEY LABEL-----"
       #define PUBKEY_HEADER         "-----BEGIN PUB KEY -----"
       #define PUBKEY_TRAILER        "-----END PUB KEY -----"
       #define NS_CERTREQ_HEADER     "-----BEGIN NEW CERTIFICATE REQUEST-----"
       #define NS_CERTREQ_TRAILER    "-----END NEW CERTIFICATE REQUEST-----"
       #define NS_CERT_ENC_HEADER    "-----BEGIN CERTIFICATE FOR ENCRYPTION-----"
       #define NS_CERT_ENC_TRAILER   "-----END CERTIFICATE FOR ENCRYPTION-----"
       #define NS_CERT_VFY_HEADER    "-----BEGIN CERTIFICATE FOR SIGNATURE VERIFICATION-----"
       #define NS_CERT_VFY_TRAILER   "-----END CERTIFICATE FOR SIGNATURE VERIFICATION-----"
       #define NS_SIG_HEADER         "-----BEGIN SIGNATURE-----"
       #define NS_SIG_TRAILER        "-----END SIGNATURE-----"
       #define NS_CERT_HEADER        "-----BEGIN CERTIFICATE-----"
       #define NS_CERT_TRAILER       "-----END CERTIFICATE-----"

      /* sample 6 commands */
       typedef enum {
           GENERATE_CSR,
           ADD_CERT_TO_DB,
           SAVE_CERT_TO_HEADER,
           ENCRYPT,
           DECRYPT,
           SIGN,
           VERIFY,
           UNKNOWN
       } CommandType;

      typedef enum {
          SYMKEY = 0,
          MACKEY = 1,
          IV     = 2,
          MAC    = 3,
          PAD    = 4,
          PUBKEY = 5,
          LAB    = 6,
          CERTENC= 7,
          CERTVFY= 8,
          SIG    = 9
       } HeaderType;


       /*
        * Print usage message and exit
        */
       static void
       Usage(const char *progName)
       {
           fprintf(stderr, "\nUsage:  %s %s %s %s %s %s %s %s %s %s\n\n",
                   progName,
                   " -<G|A|H|E|DS|V> -d <dbdirpath> ",
                   "[-p <dbpwd> | -f <dbpwdfile>] [-z <noisefilename>] [-a <\"\">]",
                   "-s <subject> -r <csr> | ",
                   "-n <nickName> -t <trust> -c <cert> [ -r <csr> -u <issuerNickname> [-x <\"\">] -m <serialNumber> ] | ",
                   "-n <nickName> -b <headerfilename> | ",
                   "-b <headerfilename> -i <ipfilename> -e <encryptfilename> | ",
                   "-b <headerfilename> -i <ipfilename> | ",
                   "-b <headerfilename> -i <ipfilename> | ",
                   "-b <headerfilename> -e <encryptfilename> -o <opfilename> \n");
           fprintf(stderr, "commands:\n\n");
           fprintf(stderr, "%s %s\n --for generating cert request (for CA also)\n\n",
                    progName, "-G -s <subject> -r <csr>");
           fprintf(stderr, "%s %s\n --to input and store cert (for CA also)\n\n",
                    progName, "-A -n <nickName> -t <trust> -c <cert> [ -r <csr> -u <issuerNickname> [-x <\"\">] -m <serialNumber> ]");
           fprintf(stderr, "%s %s\n --to put cert in header\n\n",
                    progName, "-H -n <nickname> -b <headerfilename> [-v <\"\">]");
           fprintf(stderr, "%s %s\n --to find public key from cert in header and encrypt\n\n",
                    progName, "-E -b <headerfilename> -i <ipfilename> -e <encryptfilename> ");
           fprintf(stderr, "%s %s\n --decrypt using corresponding private key \n\n",
                    progName, "-D -b <headerfilename> -e <encryptfilename> -o <opfilename>");
           fprintf(stderr, "%s %s\n --Sign using private key \n\n",
                    progName, "-S -b <headerfilename> -i <infilename> ");
           fprintf(stderr, "%s %s\n --Verify using public key \n\n",
                    progName, "-V -b <headerfilename> -i <ipfilename> ");
           fprintf(stderr, "options:\n\n");
           fprintf(stderr, "%-30s - db directory path\n\n",
                    "-d <dbdirpath>");
           fprintf(stderr, "%-30s - db password [optional]\n\n",
                    "-p <dbpwd>");
           fprintf(stderr, "%-30s - db password file [optional]\n\n",
                    "-f <dbpwdfile>");
           fprintf(stderr, "%-30s - noise file name [optional]\n\n",
                    "-z <noisefilename>");
           fprintf(stderr, "%-30s - input file name\n\n",
                    "-i <ipfilename>");
           fprintf(stderr, "%-30s - header file name\n\n",
                    "-b <headerfilename>");
           fprintf(stderr, "%-30s - encrypt file name\n\n",
                    "-e <encryptfilename>");
           fprintf(stderr, "%-30s - output file name\n\n",
                    "-o <opfilename>");
           fprintf(stderr, "%-30s - certificate serial number\n\n",
                    "-m <serialNumber>");
           fprintf(stderr, "%-30s - certificate nickname\n\n",
                    "-n <nickname>");
           fprintf(stderr, "%-30s - certificate trust\n\n",
                    "-t <trustargs>");
           fprintf(stderr, "%-30s - certificate issuer nickname\n\n",
                    "-u <issuerNickname>");
           fprintf(stderr, "%-30s - certificate signing request \n\n",
                    "-r <csr>");
           fprintf(stderr, "%-30s - generate a self-signed cert [optional]\n\n",
                    "-x");
           fprintf(stderr, "%-30s - to enable ascii [optional]\n\n",
                    "-a");
           fprintf(stderr, "%-30s - to save certificate to header file as sig verification [optional]\n\n",
                    "-v");
           exit(-1);
       }

      /*
        * Validate the options used for Generate CSR command
        */
       static void
       ValidateGenerateCSRCommand(const char *progName,
                                  const char *dbdir,
                                  CERTName   *subject,
                                  const char *subjectStr,
                                  const char *certReqFileName)
       {
           PRBool validationFailed = PR_FALSE;
           if (!subject) {
               PR_fprintf(PR_STDERR, "%s -G -d %s -s: improperly formatted name: \"%s\"\n",
                          progName, dbdir, subjectStr);
               validationFailed = PR_TRUE;
           }
           if (!certReqFileName) {
               PR_fprintf(PR_STDERR, "%s -G -d %s -s %s -r: certificate request file name not found\n",
                          progName, dbdir, subjectStr);
               validationFailed = PR_TRUE;
           }
           if (validationFailed) {
               fprintf(stderr, "\nUsage:  %s %s \n\n", progName,
                       "-G -d <dbdirpath> -s <subject> -r <csr> \n");
               exit(-1);
           }
       }

      /*
        * Validate the options used for Add Cert to DB command
        */
       static void
       ValidateAddCertToDBCommand(const char *progName,
                                  const char *dbdir,
                                  const char *nickNameStr,
                                  const char *trustStr,
                                  const char *certFileName,
                                  const char *certReqFileName,
                                  const char *issuerNameStr,
                                  const char *serialNumberStr,
                                  PRBool      selfsign)
       {
           PRBool validationFailed = PR_FALSE;
           if (!nickNameStr) {
               PR_fprintf(PR_STDERR, "%s -A -d %s -n : nick name is missing\n",
                          progName, dbdir);
               validationFailed = PR_TRUE;
           }
           if (!trustStr) {
               PR_fprintf(PR_STDERR, "%s -A -d %s -n %s -t: trust flag is missing\n",
                          progName, dbdir, nickNameStr);
               validationFailed = PR_TRUE;
           }
           if (!certFileName) {
               PR_fprintf(PR_STDERR, "%s -A -d %s -n %s -t %s -c: certificate file name not found\n",
                          progName, dbdir, nickNameStr, trustStr, serialNumberStr, certReqFileName);
               validationFailed = PR_TRUE;
           }
           if (PR_Access(certFileName, PR_ACCESS_EXISTS) == PR_FAILURE) {
               if (!certReqFileName) {
                   PR_fprintf(PR_STDERR, "%s -A -d %s -n %s -t %s -c %s -r: certificate file or certificate request file is not found\n",
                              progName, dbdir, nickNameStr, trustStr, certFileName);
                   validationFailed = PR_TRUE;
               }
               if (!selfsign && !issuerNameStr) {
                   PR_fprintf(PR_STDERR, "%s -A -d %s -n %s -t %s -c %s -r %s -u : issuer name is missing\n",
                              progName, dbdir, nickNameStr, trustStr, certFileName, certReqFileName);
                   validationFailed = PR_TRUE;
               }
               if (!serialNumberStr) {
                   PR_fprintf(PR_STDERR, "%s -A -d %s -n %s -t %s -c %s -r %s -u %s -m : serial number is missing\n",
                              progName, dbdir, nickNameStr, trustStr, certFileName, certReqFileName, issuerNameStr);
                   validationFailed = PR_TRUE;
               }
           }
           if (validationFailed) {
               fprintf(stderr, "\nUsage:  %s %s \n\n", progName,
                       " -A -d <dbdirpath> -n <nickName> -t <trust> -c <cert> \n");
               fprintf(stderr, "     OR\n");
               fprintf(stderr, "\nUsage:  %s %s \n\n", progName,
                       "-A -d <dbdirpath> -n <nickName> -t <trust> -c <cert> -r <csr> -u <issuerNickname> -m <serialNumber> [-x <\"\">] \n");
               exit(-1);
           }
       }

      /*
        * Validate the options used for Save Cert To Header command
        */
       static void
       ValidateSaveCertToHeaderCommand(const char *progName,
                                       const char *dbdir,
                                       const char *nickNameStr,
                                       const char *headerFileName)
       {
           PRBool validationFailed = PR_FALSE;
           if (!nickNameStr) {
               PR_fprintf(PR_STDERR, "%s -S -d %s -n : nick name is missing\n",
                          progName, dbdir);
               validationFailed = PR_TRUE;
           }
           if (!headerFileName) {
               PR_fprintf(PR_STDERR, "%s -S -d %s -n %s -b : header file name is not found\n",
                          progName, dbdir, nickNameStr);
               validationFailed = PR_TRUE;
           }
           if (validationFailed) {
               fprintf(stderr, "\nUsage:  %s %s \n\n", progName,
                       "-S -d <dbdirpath> -n <nickname> -b <headerfilename> [-v <\"\">]\n");
               exit(-1);
           }
       }

      /*
        * Validate the options used for Encrypt command
        */
       static void
       ValidateEncryptCommand(const char *progName,
                              const char *dbdir,
                              const char *nickNameStr,
                              const char *headerFileName,
                              const char *inFileName,
                              const char *encryptedFileName)
       {
           PRBool validationFailed = PR_FALSE;
           if (!nickNameStr) {
               PR_fprintf(PR_STDERR, "%s -E -d %s -n : nick name is missing\n",
                          progName, dbdir);
               validationFailed = PR_TRUE;
           }
           if (!headerFileName) {
               PR_fprintf(PR_STDERR, "%s -E -d %s -n %s -b : header file name is not found\n",
                          progName, dbdir, nickNameStr);
               validationFailed = PR_TRUE;
           }
           if (!inFileName) {
               PR_fprintf(PR_STDERR, "%s -E -d %s -n %s -b %s -i : input file name is not found\n",
                          progName, dbdir, nickNameStr, headerFileName);
               validationFailed = PR_TRUE;
           }
           if (!encryptedFileName) {
               PR_fprintf(PR_STDERR, "%s -E -d %s -n %s -b %s -i %s -e : encrypt file name is not found\n",
                          progName, dbdir, nickNameStr, headerFileName, inFileName);
               validationFailed = PR_TRUE;
           }
           if (validationFailed) {
               fprintf(stderr, "\nUsage:  %s %s \n\n", progName,
                       "-E -d <dbdirpath> -b <headerfilename> -i <ipfilename> -e <encryptfilename> -n <nickname> \n");
               exit(-1);
           }
       }

      /*
        * Validate the options used for Sign command
        */
       static void
       ValidateSignCommand(const char *progName,
                              const char *dbdir,
                              const char *nickNameStr,
                              const char *headerFileName,
                              const char *inFileName)
       {
           PRBool validationFailed = PR_FALSE;
           if (!nickNameStr) {
               PR_fprintf(PR_STDERR, "%s -I -d %s -n : nick name is missing\n",
                          progName, dbdir);
               validationFailed = PR_TRUE;
           }
           if (!headerFileName) {
               PR_fprintf(PR_STDERR, "%s -I -d %s -n %s -b : header file name is not found\n",
                          progName, dbdir, nickNameStr);
               validationFailed = PR_TRUE;
           }
           if (!inFileName) {
               PR_fprintf(PR_STDERR, "%s -I -d %s -n %s -b %s -i : input file name is not found\n",
                          progName, dbdir, nickNameStr, headerFileName);
               validationFailed = PR_TRUE;
           }
           if (validationFailed) {
               fprintf(stderr, "\nUsage:  %s %s \n\n", progName,
                       "-I -d <dbdirpath> -b <headerfilename> -i <ipfilename> -n <nickname> \n");
               exit(-1);
           }
       }

      /*
        * Validate the options used for verify command
        */
       static void
       ValidateVerifyCommand(const char *progName,
                              const char *dbdir,
                              const char *headerFileName,
                              const char *inFileName)
       {
           PRBool validationFailed = PR_FALSE;
           if (!headerFileName) {
               PR_fprintf(PR_STDERR, "%s -V -d %s -b : header file name is not found\n",
                          progName, dbdir);
               validationFailed = PR_TRUE;
           }
           if (!inFileName) {
               PR_fprintf(PR_STDERR, "%s -I -d %s -b %s -i : input file name is not found\n",
                          progName, dbdir, headerFileName);
               validationFailed = PR_TRUE;
           }
           if (validationFailed) {
               fprintf(stderr, "\nUsage:  %s %s \n\n", progName,
                       "-I -d <dbdirpath> -b <headerfilename> -i <ipfilename> \n");
               exit(-1);
           }
       }

      /*
        * Validate the options used for Decrypt command
        */
       static void
       ValidateDecryptCommand(const char *progName,
                              const char *dbdir,
                              const char *headerFileName,
                              const char *encryptedFileName,
                              const char *outFileName)
       {
           PRBool validationFailed = PR_FALSE;
           if (!headerFileName) {
               PR_fprintf(PR_STDERR, "%s -D -d %s -b : header file name is not found\n",
                          progName, dbdir);
               validationFailed = PR_TRUE;
           }
           if (!encryptedFileName) {
               PR_fprintf(PR_STDERR, "%s -D -d %s -b %s -e : encrypt file name is not found\n",
                          progName, dbdir, headerFileName);
               validationFailed = PR_TRUE;
           }
           if (!outFileName) {
               PR_fprintf(PR_STDERR, "%s -D -d %s -b %s -e %s -o : output file name is not found\n",
                          progName, dbdir, headerFileName, encryptedFileName);
               validationFailed = PR_TRUE;
           }
           if (validationFailed) {
               fprintf(stderr, "\nUsage:  %s %s \n\n", progName,
                       "-D -d <dbdirpath> -b <headerfilename> -e <encryptfilename> -o <opfilename>\n");
               exit(-1);
           }
       }

      /*
        * Sign the contents of input file using private key and
        * return result as SECItem
        */
       SECStatus
       SignData(const char *inFileName, SECKEYPrivateKey *pk, SECItem *res)
       {
           SECStatus     rv         = SECFailure;
           unsigned int  nb;
           unsigned char ibuf[4096];
           PRFileDesc   *inFile     = NULL;
           SGNContext   *sgn        = NULL;

          /*  Open the input file for reading */
           inFile = PR_Open(inFileName, PR_RDONLY, 0);
           if (!inFile) {
               PR_fprintf(PR_STDERR, "Unable to open \"%s\" for reading.\n",
                          inFileName);
               rv = SECFailure;
               goto cleanup;
           }

          /* Sign using private key */

          sgn = SGN_NewContext(SEC_OID_PKCS1_MD5_WITH_RSA_ENCRYPTION, pk);
           if (!sgn) {
               PR_fprintf(PR_STDERR, "unable to create context for signing\n");
               rv = SECFailure;
               goto cleanup;
           }

          rv = SGN_Begin(sgn);
           if (rv != SECSuccess) {
               PR_fprintf(PR_STDERR, "problem while SGN_Begin\n");
               goto cleanup;
           }
           while ((nb = PR_Read(inFile, ibuf, sizeof(ibuf))) > 0) {
               rv = SGN_Update(sgn, ibuf, nb);
               if (rv != SECSuccess) {
                   PR_fprintf(PR_STDERR, "problem while SGN_Update\n");
                   goto cleanup;
               }
           }
           rv = SGN_End(sgn, res);
           if (rv != SECSuccess) {
               PR_fprintf(PR_STDERR, "problem while SGN_End\n");
               goto cleanup;
           }
       cleanup:
           if (inFile) {
               PR_Close(inFile);
           }
           if (sgn) {
               SGN_DestroyContext(sgn, PR_TRUE);
           }
           return rv;
       }

      /*
        * Verify the signature using public key
        */
       SECStatus
       VerifyData(const char *inFileName, SECKEYPublicKey *pk,
                  SECItem *sigItem, secuPWData *pwdata)
       {
           unsigned int  nb;
           unsigned char ibuf[4096];
           SECStatus     rv     = SECFailure;
           VFYContext   *vfy    = NULL;
           PRFileDesc   *inFile = NULL;

          /*  Open the input file for reading */
           inFile = PR_Open(inFileName, PR_RDONLY, 0);
           if (!inFile) {
               PR_fprintf(PR_STDERR, "Unable to open \"%s\" for reading.\n",
                          inFileName);
               rv = SECFailure;
               goto cleanup;
           }

          vfy = VFY_CreateContext(pk,
                                  sigItem,
                                  SEC_OID_PKCS1_MD5_WITH_RSA_ENCRYPTION,
                                  pwdata);
           if (!vfy) {
               PR_fprintf(PR_STDERR, "unable to create context for verifying signature\n");
               rv = SECFailure;
               goto cleanup;
           }
           rv = VFY_Begin(vfy);
           if (rv != SECSuccess) {
               PR_fprintf(PR_STDERR, "problem while VFY_Begin\n");
               goto cleanup;
           }
           while ((nb = PR_Read(inFile, ibuf, sizeof(ibuf))) > 0) {
               rv = VFY_Update(vfy, ibuf, nb);
               if (rv != SECSuccess) {
                   PR_fprintf(PR_STDERR, "problem while VFY_Update\n");
                   goto cleanup;
               }
           }
           rv = VFY_End(vfy);
           if (rv != SECSuccess) {
               PR_fprintf(PR_STDERR, "problem while VFY_End\n");
               goto cleanup;
           }

      cleanup:
           if (inFile) {
               PR_Close(inFile);
           }
           if (vfy) {
               VFY_DestroyContext(vfy, PR_TRUE);
           }
           return rv;
       }

      /*
        * Write Cryptographic parameters to header file
        */
       SECStatus
       WriteToHeaderFile(const char *buf, unsigned int len, HeaderType type,
                         PRFileDesc *outFile)
       {
           SECStatus      rv;
           const char    *header;
           const char    *trailer;

          switch (type) {
           case SYMKEY:
               header = ENCKEY_HEADER;
               trailer = ENCKEY_TRAILER;
               break;
           case MACKEY:
               header =  MACKEY_HEADER;
               trailer = MACKEY_TRAILER;
               break;
           case IV:
               header = IV_HEADER;
               trailer = IV_TRAILER;
               break;
           case MAC:
               header = MAC_HEADER;
               trailer = MAC_TRAILER;
               break;
           case PAD:
               header = PAD_HEADER;
               trailer = PAD_TRAILER;
               break;
           case PUBKEY:
               header = PUBKEY_HEADER;
               trailer = PUBKEY_TRAILER;
               break;
           case CERTENC:
               header  = NS_CERT_ENC_HEADER;
               trailer = NS_CERT_ENC_TRAILER;
               break;
           case CERTVFY:
               header  = NS_CERT_VFY_HEADER;
               trailer = NS_CERT_VFY_TRAILER;
               break;
           case SIG:
               header  = NS_SIG_HEADER;
               trailer = NS_SIG_TRAILER;
               break;
           case LAB:
               header = LAB_HEADER;
               trailer = LAB_TRAILER;
               PR_fprintf(outFile, "%s\n", header);
               PR_fprintf(outFile, "%s\n", buf);
               PR_fprintf(outFile, "%s\n\n", trailer);
               return SECSuccess;
               break;
           default:
               return SECFailure;
           }

          PR_fprintf(outFile, "%s\n", header);
           PrintAsHex(outFile, buf, len);
           PR_fprintf(outFile, "%s\n\n", trailer);
           return SECSuccess;
       }

      /*
        * Read cryptographic parameters from the header file
        */
       SECStatus
       ReadFromHeaderFile(const char *fileName, HeaderType type,
                          SECItem *item, PRBool isHexData)
       {
           SECStatus      rv = SECSuccess;
           PRFileDesc*    file = NULL;
           SECItem        filedata;
           SECItem        outbuf;
           unsigned char *nonbody;
           unsigned char *body;
           char          *header;
           char          *trailer;

          outbuf.type = siBuffer;
           file = PR_Open(fileName, PR_RDONLY, 0);
           if (!file) {
               PR_fprintf(PR_STDERR, "Failed to open %s\n", fileName);
               rv = SECFailure;
               goto cleanup;
           }
           switch (type) {
           case PUBKEY:
               header = PUBKEY_HEADER;
               trailer = PUBKEY_TRAILER;
               break;
           case SYMKEY:
               header = ENCKEY_HEADER;
               trailer = ENCKEY_TRAILER;
               break;
           case MACKEY:
               header = MACKEY_HEADER;
               trailer = MACKEY_TRAILER;
               break;
           case IV:
               header = IV_HEADER;
               trailer = IV_TRAILER;
               break;
           case MAC:
               header = MAC_HEADER;
               trailer = MAC_TRAILER;
               break;
           case PAD:
               header = PAD_HEADER;
               trailer = PAD_TRAILER;
               break;
           case LAB:
               header = LAB_HEADER;
               trailer = LAB_TRAILER;
               break;
           case CERTENC:
               header  = NS_CERT_ENC_HEADER;
               trailer = NS_CERT_ENC_TRAILER;
               break;
           case CERTVFY:
               header  = NS_CERT_VFY_HEADER;
               trailer = NS_CERT_VFY_TRAILER;
               break;
           case SIG:
               header  = NS_SIG_HEADER;
               trailer = NS_SIG_TRAILER;
               break;
           default:
               rv = SECFailure;
               goto cleanup;
           }

          rv = FileToItem(&filedata, file);
           nonbody = (char *)filedata.data;
           if (!nonbody) {
               PR_fprintf(PR_STDERR, "unable to read data from input file\n");
               rv = SECFailure;
               goto cleanup;
           }

          /* check for headers and trailers and remove them */
           if ((body = strstr(nonbody, header)) != NULL) {
               char *trail = NULL;
               nonbody = body;
               body = PORT_Strchr(body, '\n');
               if (!body)
                   body = PORT_Strchr(nonbody, '\r'); /* maybe this is a MAC file */
               if (body)
                   trail = strstr(++body, trailer);
               if (trail != NULL) {
                   *trail = '\0';
               } else {
                   PR_fprintf(PR_STDERR,  "input has header but no trailer\n");
                   PORT_Free(filedata.data);
                   rv = SECFailure;
                   goto cleanup;
               }
           } else {
               /* headers didn't exist */
               char *trail = NULL;
               body = nonbody;
               if (body) {
                   trail = strstr(++body, trailer);
                   if (trail != NULL) {
                       PR_fprintf(PR_STDERR,  "input has no header but has trailer\n");
                       PORT_Free(filedata.data);
                       rv = SECFailure;
                       goto cleanup;
                   }
               }
           }
           HexToBuf(body, item, isHexData);
       cleanup:
           if (file) {
               PR_Close(file);
           }
           return rv;
       }

      /*
        * Generate the private key
        */
       SECKEYPrivateKey *
       GeneratePrivateKey(KeyType keytype, PK11SlotInfo *slot, int size,
                          int publicExponent, const char *noise,
                          SECKEYPublicKey **pubkeyp, const char *pqgFile,
                          secuPWData *pwdata)
       {
           CK_MECHANISM_TYPE  mechanism;
           SECOidTag          algtag;
           PK11RSAGenParams   rsaparams;
           void              *params;
           SECKEYPrivateKey  *privKey    = NULL;
           SECStatus          rv;
           unsigned char      randbuf[BLOCKSIZE + 1];

          rv = GenerateRandom(randbuf, BLOCKSIZE);
           if (rv != SECSuccess) {
               fprintf(stderr, "Error while generating the random numbers : %s\n",
                       PORT_ErrorToString(rv));
               goto cleanup;
           }
           PK11_RandomUpdate(randbuf, BLOCKSIZE);
           switch (keytype) {
               case rsaKey:
                   rsaparams.keySizeInBits = size;
                   rsaparams.pe            = publicExponent;
                   mechanism               = CKM_RSA_PKCS_KEY_PAIR_GEN;
                   algtag                  = SEC_OID_PKCS1_MD5_WITH_RSA_ENCRYPTION;
                   params                  = &rsaparams;
                   break;
               default:
                   goto cleanup;
           }
           fprintf(stderr, "\n\n");
           fprintf(stderr, "Generating key.  This may take a few moments...\n\n");
           privKey = PK11_GenerateKeyPair(slot, mechanism, params, pubkeyp,
                                              PR_TRUE /*isPerm*/, PR_TRUE /*isSensitive*/,
                                              pwdata);
       cleanup:
           return privKey;
       }

      /*
        * Get the certificate request from CSR
        */
       static CERTCertificateRequest *
       GetCertRequest(char *inFileName, PRBool ascii)
       {
           CERTSignedData signedData;
           SECItem reqDER;
           CERTCertificateRequest *certReq = NULL;
           SECStatus rv                    = SECSuccess;
           PRArenaPool *arena              = NULL;

          reqDER.data = NULL;
           arena = PORT_NewArena(DER_DEFAULT_CHUNKSIZE);
           if (arena == NULL) {
               rv = SECFailure;
               goto cleanup;
           }

          rv = ReadDERFromFile(&reqDER, inFileName, ascii);
           if (rv) {
               rv = SECFailure;
               goto cleanup;
           }
           certReq = (CERTCertificateRequest*) PORT_ArenaZAlloc
                      (arena, sizeof(CERTCertificateRequest));
           if (!certReq) {
               rv = SECFailure;
               goto cleanup;
           }
           certReq->arena = arena;

          /* Since cert request is a signed data, must decode to get the inner data */
           PORT_Memset(&signedData, 0, sizeof(signedData));
           rv = SEC_ASN1DecodeItem(arena, &signedData,
                                   SEC_ASN1_GET(CERT_SignedDataTemplate), &reqDER);
           if (rv) {
               rv = SECFailure;
               goto cleanup;
           }
           rv = SEC_ASN1DecodeItem(arena, certReq,
                                   SEC_ASN1_GET(CERT_CertificateRequestTemplate), &signedData.data);
           if (rv) {
               rv = SECFailure;
               goto cleanup;
           }
           rv = CERT_VerifySignedDataWithPublicKeyInfo(&signedData,
                       &certReq->subjectPublicKeyInfo, NULL /* wincx */);
           if (reqDER.data) {
               SECITEM_FreeItem(&reqDER, PR_FALSE);
           }

      cleanup:
           if (rv) {
               PR_fprintf(PR_STDERR, "bad certificate request\n");
               if (arena) {
                   PORT_FreeArena(arena, PR_FALSE);
               }
               certReq = NULL;
           }
           return certReq;
       }

      /*
        * Sign Cert
        */
       static SECItem *
       SignCert(CERTCertDBHandle *handle, CERTCertificate *cert,
                PRBool selfsign, SECOidTag hashAlgTag,
                SECKEYPrivateKey *privKey, char *issuerNickName, void *pwarg)
       {
           SECItem der;
           SECStatus rv;
           SECOidTag algID;
           void *dummy;
           PRArenaPool *arena             = NULL;
           SECItem *result                = NULL;
           SECKEYPrivateKey *caPrivateKey = NULL;

          if (!selfsign) {
               CERTCertificate *issuer = PK11_FindCertFromNickname(issuerNickName, pwarg);
               if ((CERTCertificate *)NULL == issuer) {
                   PR_fprintf(PR_STDERR, "unable to find issuer with nickname %s\n",
                              issuerNickName);
                   goto cleanup;
               }
               privKey = caPrivateKey = PK11_FindKeyByAnyCert(issuer, pwarg);
               CERT_DestroyCertificate(issuer);
               if (caPrivateKey == NULL) {
                   PR_fprintf(PR_STDERR, "unable to retrieve key  %s\n",
                              issuerNickName);
                   goto cleanup;
               }
           }
           arena = cert->arena;
           algID = SEC_GetSignatureAlgorithmOidTag(privKey->keyType, hashAlgTag);
           if (algID == SEC_OID_UNKNOWN) {
               PR_fprintf(PR_STDERR, "Unknown key or hash type for issuer.\n");
               goto cleanup;
           }
           rv = SECOID_SetAlgorithmID(arena, &cert->signature, algID, 0);
           if (rv != SECSuccess) {
               PR_fprintf(PR_STDERR, "Could not set signature algorithm id.\n%s\n",
                          PORT_ErrorToString(rv));
               goto cleanup;
           }

          /* we only deal with cert v3 here */
           *(cert->version.data) = 2;
           cert->version.len = 1;

          der.len = 0;
           der.data = NULL;
           dummy = SEC_ASN1EncodeItem (arena, &der, cert,
                                       SEC_ASN1_GET(CERT_CertificateTemplate));
           if (!dummy) {
               PR_fprintf(PR_STDERR, "Could not encode certificate.\n");
               goto cleanup;
           }

          result = (SECItem *) PORT_ArenaZAlloc (arena, sizeof (SECItem));
           if (result == NULL) {
               PR_fprintf(PR_STDERR, "Could not allocate item for certificate data.\n");
               goto cleanup;
           }

          rv = SEC_DerSignData(arena, result, der.data, der.len, privKey, algID);
           if (rv != SECSuccess) {
               PR_fprintf(PR_STDERR, "Could not sign encoded certificate data : %s\n",
                          PORT_ErrorToString(rv));
               /* result allocated out of the arena, it will be freed
                * when the arena is freed */
               result = NULL;
               goto cleanup;
           }
           cert->derCert = *result;
       cleanup:
           if (caPrivateKey) {
               SECKEY_DestroyPrivateKey(caPrivateKey);
           }
           return result;
       }

      /*
        * MakeV1Cert
        */
       static CERTCertificate *
       MakeV1Cert(CERTCertDBHandle       *handle,
                  CERTCertificateRequest *req,
                  char *                  issuerNickName,
                  PRBool                  selfsign,
                  unsigned int            serialNumber,
                  int                     warpmonths,
                  int                     validityMonths)
       {
           PRExplodedTime  printableTime;
           PRTime          now;
           PRTime          after;
           CERTValidity    *validity   = NULL;
           CERTCertificate *issuerCert = NULL;
           CERTCertificate *cert       = NULL;

          if ( !selfsign ) {
               issuerCert = CERT_FindCertByNicknameOrEmailAddr(handle, issuerNickName);
               if (!issuerCert) {
                   PR_fprintf(PR_STDERR, "could not find certificate named %s\n",
                              issuerNickName);
                   goto cleanup;
               }
           }

          now = PR_Now();
           PR_ExplodeTime (now, PR_GMTParameters, &printableTime);
           if ( warpmonths ) {
               printableTime.tm_month += warpmonths;
               now = PR_ImplodeTime (&printableTime);
               PR_ExplodeTime (now, PR_GMTParameters, &printableTime);
           }
           printableTime.tm_month += validityMonths;
           after = PR_ImplodeTime (&printableTime);

          /* note that the time is now in micro-second unit */
           validity = CERT_CreateValidity (now, after);
           if (validity) {
               cert = CERT_CreateCertificate(serialNumber,
                            (selfsign ? &req->subject : &issuerCert->subject),
                            validity, req);

              CERT_DestroyValidity(validity);
           }
       cleanup:
           if ( issuerCert ) {
               CERT_DestroyCertificate (issuerCert);
           }
           return cert;
       }

      /*
        * Add a certificate to the nss database
        */
       SECStatus
       AddCert(PK11SlotInfo *slot, CERTCertDBHandle *handle,
               const char *name, char *trusts, char *inFileName,
               PRBool ascii, PRBool emailcert, void *pwdata)
       {
           SECItem         certDER;
           SECStatus       rv;
           CERTCertTrust   *trust = NULL;
           CERTCertificate *cert = NULL;

          certDER.data = NULL;

          /* Read in the entire file specified with the -i argument */
           rv = ReadDERFromFile(&certDER, inFileName, ascii);
           if (rv != SECSuccess) {
               PR_fprintf(PR_STDERR, "unable to read input file %s : %s\n",
                          inFileName, PORT_ErrorToString(rv));
               goto cleanup;
           }

          /* Read in an ASCII cert and return a CERTCertificate */
           cert = CERT_DecodeCertFromPackage((char *)certDER.data, certDER.len);
           if (!cert) {
               PR_fprintf(PR_STDERR, "could not obtain certificate from file\n");
               rv = SECFailure;
               goto cleanup;
           }

          /* Create a cert trust */
           trust = (CERTCertTrust *)PORT_ZAlloc(sizeof(CERTCertTrust));
           if (!trust) {
               PR_fprintf(PR_STDERR, "unable to allocate cert trust\n");
               rv = SECFailure;
               goto cleanup;
           }

          rv = CERT_DecodeTrustString(trust, trusts);
           if (rv) {
               PR_fprintf(PR_STDERR, "unable to decode trust string\n");
               rv = SECFailure;
               goto cleanup;
           }

          rv =  PK11_ImportCert(slot, cert, CK_INVALID_HANDLE, name, PR_FALSE);
           if (rv != SECSuccess) {
               /* sigh, PK11_Import Cert and CERT_ChangeCertTrust should have
                * been coded to take a password arg. */
               if (PORT_GetError() == SEC_ERROR_TOKEN_NOT_LOGGED_IN) {
                   rv = PK11_Authenticate(slot, PR_TRUE, pwdata);
                   if (rv != SECSuccess) {
                       PR_fprintf(PR_STDERR, "could not authenticate to token  %s : %s\n",
                                  PK11_GetTokenName(slot), PORT_ErrorToString(rv));
                       rv = SECFailure;
                       goto cleanup;
                   }
                   rv = PK11_ImportCert(slot, cert, CK_INVALID_HANDLE,
                                        name, PR_FALSE);
               }
               if (rv != SECSuccess) {
                   PR_fprintf(PR_STDERR,
                              "could not add certificate to token or database : %s\n",
                              PORT_ErrorToString(rv));
                   rv = SECFailure;
                   goto cleanup;
               }
           }
           rv = CERT_ChangeCertTrust(handle, cert, trust);
           if (rv != SECSuccess) {
               if (PORT_GetError() == SEC_ERROR_TOKEN_NOT_LOGGED_IN) {
                   rv = PK11_Authenticate(slot, PR_TRUE, pwdata);
                   if (rv != SECSuccess) {
                       PR_fprintf(PR_STDERR, "could not authenticate to token  %s : %s\n",
                                  PK11_GetTokenName(slot), PORT_ErrorToString(rv));
                       rv = SECFailure;
                       goto cleanup;
                   }
                   rv = CERT_ChangeCertTrust(handle, cert, trust);
               }
               if (rv != SECSuccess) {
                   PR_fprintf(PR_STDERR, "could not change trust on certificate : %s\n",
                              PORT_ErrorToString(rv));
                   rv = SECFailure;
                   goto cleanup;
               }
           }

          if (emailcert) {
               CERT_SaveSMimeProfile(cert, NULL, pwdata);
           }

      cleanup:
           if (cert) {
               CERT_DestroyCertificate (cert);
           }
           if (trust) {
               PORT_Free(trust);
           }
           if (certDER.data) {
               PORT_Free(certDER.data);
           }
           return rv;
       }

      /*
        * Create a certificate
        */
       static SECStatus
       CreateCert(
               CERTCertDBHandle *handle,
               PK11SlotInfo *slot,
               char *  issuerNickName,
               char *inFileName,
               char *outFileName,
               SECKEYPrivateKey **selfsignprivkey,
               void    *pwarg,
               SECOidTag hashAlgTag,
               unsigned int serialNumber,
               int     warpmonths,
               int     validityMonths,
               const char *dnsNames,
               PRBool  ascii,
               PRBool  selfsign)
       {
           void                   *extHandle;
           SECItem                reqDER;
           CERTCertExtension      **CRexts;
           SECStatus              rv               = SECSuccess;
           CERTCertificate        *subjectCert     = NULL;
           CERTCertificateRequest *certReq         = NULL;
           PRFileDesc             *outFile         = NULL;
           SECItem                *certDER         = NULL;

          reqDER.data = NULL;
           outFile = PR_Open(outFileName,
                             PR_RDWR | PR_CREATE_FILE | PR_TRUNCATE, 00660);

          /* Create a cert request object from the input cert request der */
           certReq = GetCertRequest(inFileName, ascii);
           if (certReq == NULL) {
               rv = SECFailure;
               goto cleanup;
           }
           subjectCert = MakeV1Cert(handle, certReq, issuerNickName, selfsign,
                                    serialNumber, warpmonths, validityMonths);
           if (subjectCert == NULL) {
               rv = SECFailure;
               goto cleanup;
           }

          extHandle = CERT_StartCertExtensions (subjectCert);
           if (extHandle == NULL) {
               rv = SECFailure;
               goto cleanup;
           }

          if (certReq->attributes != NULL &&
               certReq->attributes[0] != NULL &&
               certReq->attributes[0]->attrType.data != NULL &&
               certReq->attributes[0]->attrType.len   > 0    &&
               SECOID_FindOIDTag(&certReq->attributes[0]->attrType)
                       == SEC_OID_PKCS9_EXTENSION_REQUEST) {
               rv = CERT_GetCertificateRequestExtensions(certReq, &CRexts);
               if (rv != SECSuccess) {
                   PR_fprintf(PR_STDERR, "%s\n", PORT_ErrorToString(rv));
                   goto cleanup;
               }
               rv = CERT_MergeExtensions(extHandle, CRexts);
               if (rv != SECSuccess) {
                   PR_fprintf(PR_STDERR, "%s\n", PORT_ErrorToString(rv));
                   goto cleanup;
               }
           }

          CERT_FinishExtensions(extHandle);

          /* self-signing a cert request, find the private key */
           if (*selfsignprivkey == NULL) {
               *selfsignprivkey = PK11_FindKeyByDERCert(slot, subjectCert, pwarg);
               if (!*selfsignprivkey) {
                   PR_fprintf(PR_STDERR, "Failed to locate private key.\n");
                   rv = SECFailure;
                   goto cleanup;
               }
           }

          certDER = SignCert(handle, subjectCert, selfsign, hashAlgTag,
                              *selfsignprivkey, issuerNickName,pwarg);
           if (certDER) {
               if (ascii) {
                   PR_fprintf(outFile, "%s\n%s\n%s\n", NS_CERT_HEADER,
                              BTOA_DataToAscii(certDER->data, certDER->len),
                              NS_CERT_TRAILER);
               } else {
                   PR_Write(outFile, certDER->data, certDER->len);
               }
           }
           if (rv != SECSuccess) {
               PRErrorCode  perr = PR_GetError();
               PR_fprintf(PR_STDERR, "unable to create cert %s\n",
                          perr);
           }
       cleanup:
           if (outFile) {
               PR_Close(outFile);
           }
           if (*selfsignprivkey) {
               SECKEY_DestroyPrivateKey(*selfsignprivkey);
           }
           if (certReq) {
               CERT_DestroyCertificateRequest(certReq);
           }
           if (subjectCert) {
               CERT_DestroyCertificate(subjectCert);
           }
           return rv;
       }

      /*
        *  Generate the certificate request with subject
        */
       static SECStatus
       CertReq(SECKEYPrivateKey *privk, SECKEYPublicKey *pubk, KeyType keyType,
               SECOidTag hashAlgTag, CERTName *subject, PRBool ascii,
               const char *certReqFileName)
       {
           SECOidTag                 signAlgTag;
           SECItem                   result;
           PRInt32                   numBytes;
           SECStatus                 rv            = SECSuccess;
           PRArenaPool              *arena         = NULL;
           void                     *extHandle     = NULL;
           PRFileDesc               *outFile       = NULL;
           CERTSubjectPublicKeyInfo *spki          = NULL;
           CERTCertificateRequest   *cr            = NULL;
           SECItem                  *encoding      = NULL;

          /* If the certificate request file already exists, delete it */
           if (PR_Access(certReqFileName, PR_ACCESS_EXISTS) == PR_SUCCESS) {
               PR_Delete(certReqFileName);
           }
           /*  Open the certificate request file to write */
           outFile = PR_Open(certReqFileName, PR_CREATE_FILE | PR_RDWR | PR_TRUNCATE, 00660);
           if (!outFile) {
               PR_fprintf(PR_STDERR,
                          "unable to open \"%s\" for writing (%ld, %ld).\n",
                          certReqFileName, PR_GetError(), PR_GetOSError());
               goto cleanup;
           }
           /* Create info about public key */
           spki = SECKEY_CreateSubjectPublicKeyInfo(pubk);
           if (!spki) {
               PR_fprintf(PR_STDERR, "unable to create subject public key\n");
               rv = SECFailure;
               goto cleanup;
           }

          /* Generate certificate request */
           cr = CERT_CreateCertificateRequest(subject, spki, NULL);
           if (!cr) {
               PR_fprintf(PR_STDERR, "unable to make certificate request\n");
               rv = SECFailure;
               goto cleanup;
           }

           arena = PORT_NewArena(DER_DEFAULT_CHUNKSIZE);
           if (!arena) {
               fprintf(stderr, "out of memory");
               rv = SECFailure;
               goto cleanup;
           }

          extHandle = CERT_StartCertificateRequestAttributes(cr);
           if (extHandle == NULL) {
               PORT_FreeArena (arena, PR_FALSE);
               rv = SECFailure;
               goto cleanup;
           }

          CERT_FinishExtensions(extHandle);
           CERT_FinishCertificateRequestAttributes(cr);

           /* Der encode the request */
           encoding = SEC_ASN1EncodeItem(arena, NULL, cr,
                                         SEC_ASN1_GET(CERT_CertificateRequestTemplate));
           if (encoding == NULL) {
               PR_fprintf(PR_STDERR, "der encoding of request failed\n");
               rv = SECFailure;
               goto cleanup;
           }

          /* Sign the request */
           signAlgTag = SEC_GetSignatureAlgorithmOidTag(keyType, hashAlgTag);
           if (signAlgTag == SEC_OID_UNKNOWN) {
               PR_fprintf(PR_STDERR, "unknown Key or Hash type\n");
               rv = SECFailure;
           goto cleanup;
           }
           rv = SEC_DerSignData(arena, &result, encoding->data, encoding->len,
                                privk, signAlgTag);
           if (rv) {
               PR_fprintf(PR_STDERR, "signing of data failed\n");
               rv = SECFailure;
               goto cleanup;
           }

          /* Encode request in specified format */
           if (ascii) {
               char *obuf;
               char *name, *email, *org, *state, *country;
               SECItem *it;
               int total;

              it = &result;

              obuf = BTOA_ConvertItemToAscii(it);
               total = PL_strlen(obuf);

              name = CERT_GetCommonName(subject);
               if (!name) {
                   name = strdup("(not specified)");
               }

              email = CERT_GetCertEmailAddress(subject);
               if (!email)
                   email = strdup("(not specified)");

              org = CERT_GetOrgName(subject);
               if (!org)
                   org = strdup("(not specified)");

              state = CERT_GetStateName(subject);
               if (!state)
                   state = strdup("(not specified)");

              country = CERT_GetCountryName(subject);
               if (!country)
                   country = strdup("(not specified)");

              PR_fprintf(outFile,
                          "\nCertificate request generated by Netscape certutil\n");
               PR_fprintf(outFile, "Common Name: %s\n", name);
               PR_fprintf(outFile, "Email: %s\n", email);
               PR_fprintf(outFile, "Organization: %s\n", org);
               PR_fprintf(outFile, "State: %s\n", state);
               PR_fprintf(outFile, "Country: %s\n\n", country);

              PR_fprintf(outFile, "%s\n", NS_CERTREQ_HEADER);
               numBytes = PR_Write(outFile, obuf, total);
               if (numBytes != total) {
                   PR_fprintf(PR_STDERR, "write error\n");
                   return SECFailure;
               }
               PR_fprintf(outFile, "\n%s\n", NS_CERTREQ_TRAILER);
           } else {
               numBytes = PR_Write(outFile, result.data, result.len);
               if (numBytes != (int)result.len) {
                   PR_fprintf(PR_STDERR, "write error\n");
                   rv = SECFailure;
                   goto cleanup;
               }
           }
       cleanup:
           if (outFile) {
               PR_Close(outFile);
           }
           if (privk) {
               SECKEY_DestroyPrivateKey(privk);
           }
           if (pubk) {
               SECKEY_DestroyPublicKey(pubk);
           }
           return rv;
       }

      /*
        * Create certificate request with subject
        */
       SECStatus CreateCertRequest(PK11SlotInfo *slot,
           secuPWData   *pwdata,
           CERTName     *subject,
           char   *certReqFileName,
           PRBool       ascii)
       {
           SECStatus rv;
           SECKEYPrivateKey    *privkey         = NULL;
           SECKEYPublicKey     *pubkey          = NULL;
           KeyType             keytype          = rsaKey;
           int                 keysize          = DEFAULT_KEY_BITS;
           int                 publicExponent   = 0x010001;
           SECOidTag           hashAlgTag       = SEC_OID_UNKNOWN;

          privkey = GeneratePrivateKey(keytype, slot, keysize,
                                        publicExponent, NULL,
                                        &pubkey, NULL, pwdata);
           if (privkey == NULL) {
               PR_fprintf(PR_STDERR, "unable to generate key(s)\n");
               rv = SECFailure;
               goto cleanup;
           }
           privkey->wincx = pwdata;
           PORT_Assert(pubkey != NULL);
           rv = CertReq(privkey, pubkey, keytype, hashAlgTag, subject,
                        ascii, certReqFileName);

           if (rv != SECSuccess) {
               PR_fprintf(PR_STDERR, "Failed to create Certificate Request\n");
           }
       cleanup:
           return rv;
       }

      /*
        * Creates the certificate using CSR and adds the certificate to DB
        */
       SECStatus AddCertificateToDB(PK11SlotInfo     *slot,
                                    secuPWData       *pwdata,
                                    char             *certReqFileName,
                                    char             *certFileName,
                                    char             *issuerNameStr,
                                    CERTCertDBHandle *certHandle,
                                    const char       *nickNameStr,
                                    char             *trustStr,
                                    unsigned int     serialNumber,
                                    PRBool           selfsign,
                                    PRBool           ascii)
       {
           SECStatus rv;
           SECKEYPrivateKey    *privkey         = NULL;
           SECKEYPublicKey     *pubkey          = NULL;
           SECOidTag           hashAlgTag       = SEC_OID_UNKNOWN;

          if (PR_Access(certFileName, PR_ACCESS_EXISTS) == PR_FAILURE) {
               rv = CreateCert(certHandle, slot, issuerNameStr,
                               certReqFileName, certFileName, &privkey, &pwdata, hashAlgTag,
                               serialNumber, 0, 3, NULL, ascii, selfsign);
               if (rv != SECSuccess) {
                   PR_fprintf(PR_STDERR, "Failed to create Certificate\n");
                   goto cleanup;
               }
           }
           rv = AddCert(slot, certHandle, nickNameStr,
                        trustStr, certFileName, ascii, 0, &pwdata);
           if (rv != SECSuccess) {
               PR_fprintf(PR_STDERR, "Failed to add Certificate\n");
           }
       cleanup:
           return rv;
       }

      /*
        * Finds the certificate using nickname and saves it to the header file
        */
       SECStatus AddCertificateToHeader(PK11SlotInfo     *slot,
                                        secuPWData       *pwdata,
                                        const char       *headerFileName,
                                        CERTCertDBHandle *certHandle,
                                        const char       *nickNameStr,
                                        PRBool           sigVerify)

       {
           SECStatus            rv              = SECSuccess;
           PRFileDesc          *headerFile      = NULL;
           CERTCertificate     *cert            = NULL;
           HeaderType           hType           = CERTENC;

          /* If the intermediate header file already exists, delete it */
           if (PR_Access(headerFileName, PR_ACCESS_EXISTS) == PR_SUCCESS) {
               PR_Delete(headerFileName);
           }
           headerFile = PR_Open(headerFileName, PR_CREATE_FILE | PR_RDWR | PR_TRUNCATE, 00660);
           if (!headerFile) {
               PR_fprintf(PR_STDERR,
               "unable to open \"%s\" for writing (%ld, %ld).\n",
               headerFileName, PR_GetError(), PR_GetOSError());
               rv = SECFailure;
               goto cleanup;
           }
           cert = CERT_FindCertByNicknameOrEmailAddr(certHandle, nickNameStr);
           if (!cert) {
               PR_fprintf(PR_STDERR, "could not obtain certificate from file\n");
               rv = SECFailure;
               goto cleanup;
           }
           if (sigVerify) {
               hType = CERTVFY;
           }
           WriteToHeaderFile(cert->derCert.data, cert->derCert.len, hType, headerFile);
       cleanup:
           if (headerFile) {
               PR_Close(headerFile);
           }
           if (cert) {
               CERT_DestroyCertificate(cert);
           }
           return rv;
       }

      /*
        * Finds the public key from the certificate saved in the header file
        * and encrypts with it the contents of inFileName to encryptedFileName.
        */
       SECStatus FindKeyAndEncrypt(PK11SlotInfo *slot,
                                   secuPWData *pwdata,
                                   const char *headerFileName,
                                   const char *encryptedFileName,
                                   const char *inFileName)
       {
           SECStatus           rv;
           PRFileDesc          *headerFile      = NULL;
           PRFileDesc          *encFile         = NULL;
           PRFileDesc          *inFile          = NULL;
           CERTCertificate     *cert            = NULL;
           SECItem             data;
           unsigned char       ptext[MODBLOCKSIZE];
           unsigned char       encBuf[MODBLOCKSIZE];
           unsigned int        ptextLen;
           int                 index;
           unsigned int        nWritten;
           unsigned int        pad[1];
           SECItem             padItem;
           unsigned int        paddingLength    = 0;
           SECKEYPublicKey     *pubkey          = NULL;

          /* If the intermediate encrypted file already exists, delete it*/
           if (PR_Access(encryptedFileName, PR_ACCESS_EXISTS) == PR_SUCCESS) {
               PR_Delete(encryptedFileName);
           }

          /* Read certificate from header file */
           rv = ReadFromHeaderFile(headerFileName, CERTENC, &data, PR_TRUE);
           if (rv != SECSuccess) {
               PR_fprintf(PR_STDERR, "Could not read certificate from header file\n");
               goto cleanup;
           }
           /* Read in an ASCII cert and return a CERTCertificate */
           cert = CERT_DecodeCertFromPackage((char *)data.data, data.len);
           if (!cert) {
               PR_fprintf(PR_STDERR, "could not obtain certificate from file\n");
               rv = SECFailure;
               goto cleanup;
           }
           /* Extract the public key from certificate */
           pubkey = CERT_ExtractPublicKey(cert);
           if (!pubkey) {
               PR_fprintf(PR_STDERR, "could not get key from certificate\n");
               rv = SECFailure;
               goto cleanup;
           }

          /*  Open the encrypted file for writing */
           encFile = PR_Open(encryptedFileName,
                             PR_CREATE_FILE | PR_TRUNCATE | PR_RDWR, 00660);
           if (!encFile) {
               PR_fprintf(PR_STDERR,
                          "Unable to open \"%s\" for writing.\n",
                          encryptedFileName);
               rv = SECFailure;
               goto cleanup;
           }

          /*  Open the input file for reading */
           inFile = PR_Open(inFileName, PR_RDONLY, 0);
           if (!inFile) {
               PR_fprintf(PR_STDERR, "Unable to open \"%s\" for reading.\n",
                          inFileName);
               rv = SECFailure;
               goto cleanup;
           }

          /*  Open the header file to write padding */
           headerFile = PR_Open(headerFileName, PR_CREATE_FILE | PR_RDWR | PR_APPEND, 00660);
           if (!headerFile) {
               PR_fprintf(PR_STDERR, "Unable to open \"%s\" for writing.\n",
                          headerFileName);
               rv = SECFailure;
               goto cleanup;
           }

           /* Read input file  */
           while ((ptextLen = PR_Read(inFile, ptext, sizeof(ptext))) > 0) {
               if (ptextLen != MODBLOCKSIZE) {
                   paddingLength = MODBLOCKSIZE - ptextLen;
                   for ( index=0; index < paddingLength; index++) {
                       ptext[ptextLen+index] = (unsigned char)paddingLength;
                   }
                   ptextLen = MODBLOCKSIZE;
                }
                rv = PK11_PubEncryptRaw(pubkey, encBuf, ptext, ptextLen, NULL);
                nWritten = PR_Write(encFile, encBuf, ptextLen);
           }

          /* Write the padding to header file */
           pad[0] = paddingLength;
           padItem.type = siBuffer;
           padItem.data = (unsigned char *)pad;
           padItem.len  = sizeof(pad[0]);
           WriteToHeaderFile(padItem.data, padItem.len, PAD, headerFile);

      cleanup:
           if (headerFile) {
               PR_Close(headerFile);
           }
           if (encFile) {
               PR_Close(encFile);
           }
           if (inFile) {
               PR_Close(inFile);
           }
           if (pubkey) {
               SECKEY_DestroyPublicKey(pubkey);
           }
           if (cert) {
               CERT_DestroyCertificate(cert);
           }
           return rv;
       }

      /*
        * Finds the private key from db and signs the contents
        * of inFileName and writes to signatureFileName
        */
       SECStatus FindKeyAndSign(PK11SlotInfo *slot,
                                CERTCertDBHandle* certHandle,
                                secuPWData *pwdata,
                                const char *nickNameStr,
                                const char *headerFileName,
                                const char *inFileName)
       {
           SECStatus           rv;
           PRFileDesc          *headerFile      = NULL;
           PRFileDesc          *inFile          = NULL;
           CERTCertificate     *cert            = NULL;
           unsigned int        signatureLen     = 0;
           SECKEYPrivateKey    *privkey         = NULL;
           SECItem             sigItem;
           SECOidTag           hashOIDTag;

           /*  Open the header file to write padding */
           headerFile = PR_Open(headerFileName, PR_CREATE_FILE | PR_RDWR | PR_APPEND, 00660);
           if (!headerFile) {
               PR_fprintf(PR_STDERR, "Unable to open \"%s\" for writing.\n",
                          headerFileName);
               rv = SECFailure;
               goto cleanup;
           }

          /* Get the certificate by nick name  and write to header file */
           cert = CERT_FindCertByNicknameOrEmailAddr(certHandle, nickNameStr);
           if (!cert) {
               PR_fprintf(PR_STDERR, "could not obtain certificate by name - %s\n", nickNameStr);
               rv = SECFailure;
               goto cleanup;
           }
           WriteToHeaderFile(cert->derCert.data, cert->derCert.len, CERTVFY, headerFile);


           /* Find private key from certificate  */
           privkey = PK11_FindKeyByAnyCert(cert, NULL);
           if (privkey == NULL) {
               fprintf(stderr, "Couldn't find private key for cert\n");
               rv = SECFailure;
               goto cleanup;
           }

           /* Sign the contents of the input file */
           rv = SignData(inFileName, privkey, &sigItem);
           if (rv != SECSuccess) {
               PR_fprintf(PR_STDERR, "could not sign the contents from file - %s \n", inFileName);
               goto cleanup;
           }

          /* write signature to header file */
           WriteToHeaderFile(sigItem.data, sigItem.len, SIG, headerFile);

      cleanup:
           if (headerFile) {
               PR_Close(headerFile);
           }
           if (privkey) {
               SECKEY_DestroyPrivateKey(privkey);
           }
           if (cert) {
               CERT_DestroyCertificate(cert);
           }
           return rv;
       }

      /*
        * Finds the public key from certificate and verifies signature
        */
       SECStatus FindKeyAndVerify(PK11SlotInfo *slot,
                                CERTCertDBHandle* certHandle,
                                secuPWData *pwdata,
                                const char *headerFileName,
                                const char *inFileName)
       {
           SECStatus           rv               = SECFailure;
           PRFileDesc          *headerFile      = NULL;
           PRFileDesc          *inFile          = NULL;
           CERTCertificate     *cert            = NULL;
           SECKEYPublicKey     *pubkey          = NULL;
           SECItem             sigItem;
           SECItem             certData;


          /* Open the input file  */
           inFile = PR_Open(inFileName, PR_RDONLY, 0);
           if (!inFile) {
               PR_fprintf(PR_STDERR,
                          "Unable to open \"%s\" for reading.\n",
                          inFileName);
               rv = SECFailure;
               goto cleanup;
           }

          /* Open the header file to read the certificate and signature */
           headerFile = PR_Open(headerFileName, PR_RDONLY, 0);
           if (!headerFile) {
               PR_fprintf(PR_STDERR, "Unable to open \"%s\" for writing.\n",
                          headerFileName);
               rv = SECFailure;
               goto cleanup;
           }

          /* Read certificate from header file */
           rv = ReadFromHeaderFile(headerFileName, CERTVFY, &certData, PR_TRUE);
           if (rv != SECSuccess) {
               PR_fprintf(PR_STDERR, "Could not read certificate from header file\n");
               goto cleanup;
           }

          /* Read in an ASCII cert and return a CERTCertificate */
           cert = CERT_DecodeCertFromPackage((char *)certData.data, certData.len);
           if (!cert) {
               PR_fprintf(PR_STDERR, "could not obtain certificate from file\n");
               rv = SECFailure;
               goto cleanup;
           }

          /* Extract the public key from certificate */
           pubkey = CERT_ExtractPublicKey(cert);
           if (!pubkey) {
               PR_fprintf(PR_STDERR, "Could not get key from certificate\n");
               rv = SECFailure;
               goto cleanup;
           }

          /* Read signature from header file */
           rv = ReadFromHeaderFile(headerFileName, SIG, &sigItem, PR_TRUE);
           if (rv != SECSuccess) {
               PR_fprintf(PR_STDERR, "Could not read signature from header file\n");
               goto cleanup;
           }

           /* Verify with the public key */
           rv = VerifyData(inFileName, pubkey, &sigItem, pwdata);
           if (rv != SECSuccess) {
               PR_fprintf(PR_STDERR, "Couldn't verify the signature for file - %s\n", inFileName);
               goto cleanup;
           }

      cleanup:
           if (headerFile) {
               PR_Close(headerFile);
           }
           if (pubkey) {
               SECKEY_DestroyPublicKey(pubkey);
           }
           if (cert) {
               CERT_DestroyCertificate(cert);
           }
           return rv;
       }

      /*
        * Finds the private key corresponding to the certificate saved in the header file
        * and decrypts with it the contents of encryptedFileName to outFileName.
        */
       SECStatus FindKeyAndDecrypt(PK11SlotInfo *slot,
                                   secuPWData *pwdata,
                                   const char *headerFileName,
                                   const char *encryptedFileName,
                                   const char *outFileName)
       {
           SECStatus           rv;
           PRFileDesc          *encFile        = NULL;
           PRFileDesc          *outFile        = NULL;
           SECKEYPrivateKey    *pvtkey         = NULL;
           unsigned int        inFileLength    = 0;
           unsigned int        paddingLength   = 0;
           unsigned int        count           = 0;
           unsigned int        temp            = 0;
           unsigned char       ctext[MODBLOCKSIZE];
           unsigned char       decBuf[MODBLOCKSIZE];
           unsigned int        ctextLen;
           unsigned int        decBufLen;
           SECItem             padItem;
           SECItem             data;
           SECItem             signature;
           CERTCertificate     *cert            = NULL;

          /* Read certificate from header file */
           rv = ReadFromHeaderFile(headerFileName, CERTENC, &data, PR_TRUE);
           if (rv != SECSuccess) {
               PR_fprintf(PR_STDERR, "Could not read certificate from header file\n");
               goto cleanup;
           }

          /* Read padding from header file */
           rv = ReadFromHeaderFile(headerFileName, PAD, &padItem, PR_TRUE);
           if (rv != SECSuccess) {
               PR_fprintf(PR_STDERR,
                       "Could not retrieve PAD detail from header file\n");
               goto cleanup;
           }
           paddingLength = (unsigned int)padItem.data[0];
           inFileLength = FileSize(encryptedFileName);

          /* Read in an ASCII cert and return a CERTCertificate */
           cert = CERT_DecodeCertFromPackage((char *)data.data, data.len);
           if (!cert) {
               PR_fprintf(PR_STDERR, "could not obtain certificate from file\n");
               rv = SECFailure;
               goto cleanup;
           }

          /* Find private key from certificate  */
           pvtkey = PK11_FindKeyByAnyCert(cert, NULL);
           if (pvtkey == NULL) {
               fprintf(stderr, "Couldn't find private key for cert\n");
               rv = SECFailure;
               goto cleanup;
           }

          /* Open the out file to write */
           outFile = PR_Open(outFileName,
                             PR_CREATE_FILE | PR_TRUNCATE | PR_RDWR, 00660);
           if (!outFile) {
               PR_fprintf(PR_STDERR, "Unable to open \"%s\" for writing.\n",
                          outFileName);
               rv = SECFailure;
               goto cleanup;
           }
           /* Open the encrypted file for reading */
           encFile = PR_Open(encryptedFileName, PR_RDONLY, 0);
           if (!encFile) {
               PR_fprintf(PR_STDERR, "Unable to open \"%s\" for reading.\n",
                          encryptedFileName);
               rv = SECFailure;
               goto cleanup;
           }
           /* Read the encrypt file, decrypt and write to out file */
           while ((ctextLen = PR_Read(encFile, ctext, sizeof(ctext))) > 0) {
               count += ctextLen;
               rv = PK11_PubDecryptRaw(pvtkey, decBuf, &decBufLen, sizeof(decBuf), ctext, ctextLen);
               if (rv != SECSuccess) {
                   fprintf(stderr, "Couldn't decrypt\n");
                   goto cleanup;
               }
               if (decBufLen == 0) {
                   break;
               }
               if (count == inFileLength) {
                   decBufLen = decBufLen - paddingLength;
               }
               /* write the plain text to out file */
               temp = PR_Write(outFile, decBuf, decBufLen);
               if (temp != decBufLen) {
                   PR_fprintf(PR_STDERR, "write error\n");
                   rv = SECFailure;
                   break;
               }
            }
       cleanup:
           if (encFile) {
               PR_Close(encFile);
           }
           if (outFile) {
               PR_Close(outFile);
           }
           if (pvtkey) {
               SECKEY_DestroyPrivateKey(pvtkey);
           }
           if (cert) {
               CERT_DestroyCertificate(cert);
           }
           return rv;
       }

      /* Map option letter to command */
       static CommandType option2Command(char c)
       {
           switch (c) {
           case 'G': return GENERATE_CSR;
           case 'A': return ADD_CERT_TO_DB;
           case 'H': return SAVE_CERT_TO_HEADER;
           case 'E': return ENCRYPT;
           case 'D': return DECRYPT;
           case 'S': return SIGN;
           case 'V': return VERIFY;
           default:  return UNKNOWN;
           }
       }

      /*
        * This example illustrates basic encryption/decryption and MACing
        * Generates the RSA key pair as token object and outputs public key as cert request.
        * Reads cert request file and stores certificate in DB.
        * Input, store and trust CA certificate.
        * Write certificate to intermediate header file
        * Extract public key from certificate, encrypts the input file and write to external file.
        * Finds the matching private key, decrypts and write to external file
        *
        * How this sample is different from sample 5 ?
        *
        * 1. As in sample 5, output is a PKCS#10 CSR
        * 2. Input and store a cert in cert DB and also used to input, store and trust CA cert.
        * 3. Like sample 5, but puts cert in header
        * 4. Like sample 5, but finds key matching cert in header
       */
       int
       main(int argc, char **argv)
       {
           SECStatus           rv;
           PLOptState          *optstate;
           PLOptStatus         status;
           PRBool              initialized             = PR_FALSE;

          CommandType         cmd                     = UNKNOWN;
           const char          *dbdir                  = NULL;
           secuPWData          pwdata                  = { PW_NONE, 0 };

          char                *subjectStr             = NULL;
           CERTName            *subject                = 0;

          unsigned int        serialNumber            = 0;
           char                *serialNumberStr        = NULL;
           char                *trustStr               = NULL;
           CERTCertDBHandle    *certHandle;
           const char          *nickNameStr            = NULL;
           char                *issuerNameStr          = NULL;
           PRBool              selfsign                = PR_FALSE;
           PRBool              ascii                   = PR_FALSE;
           PRBool              sigVerify               = PR_FALSE;

           const char          *headerFileName         = NULL;
           const char          *encryptedFileName      = NULL;
           const char          *inFileName             = NULL;
           const char          *outFileName            = NULL;
           char                *certReqFileName        = NULL;
           char                *certFileName           = NULL;
           const char          *noiseFileName          = NULL;
           PK11SlotInfo        *slot                   = NULL;

          char * progName = strrchr(argv[0], '/');
           progName = progName ? progName + 1 : argv[0];

          /* Parse command line arguments */
           optstate = PL_CreateOptState(argc, argv, "GAHEDSVad:i:o:f:p:z:s:r:n:x:m:t:c:u:e:b:v:");
           while ((status = PL_GetNextOpt(optstate)) == PL_OPT_OK) {
               switch (optstate->option) {
               case 'a':
                   ascii = PR_TRUE;
                   break;
               case 'G':   /* Generate a CSR */
               case 'A':   /* Add cert to database */
               case 'H':   /* Save cert to the header file */
               case 'E':   /* Encrypt with public key from cert in header file */
               case 'S':   /* Sign with private key */
               case 'D':   /* Decrypt with the matching private key */
               case 'V':   /* Verify with the matching public key */
                   cmd = option2Command(optstate->option);
                   break;
               case 'd':
                   dbdir = strdup(optstate->value);
                   break;
               case 'f':
                   pwdata.source = PW_FROMFILE;
                   pwdata.data = strdup(optstate->value);
                   break;
               case 'p':
                   pwdata.source = PW_PLAINTEXT;
                   pwdata.data = strdup(optstate->value);
                   break;
               case 'i':
                   inFileName = strdup(optstate->value);
                   break;
               case 'b':
                   headerFileName = strdup(optstate->value);
                   break;
               case 'e':
                   encryptedFileName = strdup(optstate->value);
                   break;
               case 'o':
                   outFileName = strdup(optstate->value);
                   break;
               case 'z':
                   noiseFileName = strdup(optstate->value);
                   break;
               case 's':
                   subjectStr  = strdup(optstate->value);
                   subject     = CERT_AsciiToName(subjectStr);
                   break;
               case 'r':
                   certReqFileName = strdup(optstate->value);
                   break;
               case 'c':
                   certFileName = strdup(optstate->value);
                   break;
               case 'u':
                   issuerNameStr = strdup(optstate->value);
                   break;
               case 'n':
                   nickNameStr = strdup(optstate->value);
                   break;
               case 'x':
                   selfsign = PR_TRUE;
                   break;
               case 'm':
                   serialNumberStr = strdup(optstate->value);
                   serialNumber    = atoi(serialNumberStr);
                   break;
               case 't':
                   trustStr = strdup(optstate->value);
                   break;
               case 'v':
                   sigVerify = PR_TRUE;
                   break;
               default:
                   Usage(progName);
                   break;
               }
           }
           PL_DestroyOptState(optstate);

          if (cmd == UNKNOWN || !dbdir)
               Usage(progName);

          /* Open DB for read/write and authenticate to it */
           PR_Init(PR_USER_THREAD, PR_PRIORITY_NORMAL, 0);
           initialized = PR_TRUE;
           rv = NSS_InitReadWrite(dbdir);
           if (rv != SECSuccess) {
               PR_fprintf(PR_STDERR, "NSS_InitReadWrite Failed\n");
               goto cleanup;
           }

          PK11_SetPasswordFunc(GetModulePassword);
           slot = PK11_GetInternalKeySlot();
           if (PK11_NeedLogin(slot)) {
               rv = PK11_Authenticate(slot, PR_TRUE, &pwdata);
               if (rv != SECSuccess) {
                   PR_fprintf(PR_STDERR, "Could not authenticate to token %s.\n",
                              PK11_GetTokenName(slot));
                   goto cleanup;
               }
           }

          switch (cmd) {
           case GENERATE_CSR:
               ValidateGenerateCSRCommand(progName, dbdir, subject, subjectStr,
                                          certReqFileName);
               /* Generate a CSR */
               rv = CreateCertRequest(slot, &pwdata, subject,
                                      certReqFileName, ascii);
               if (rv != SECSuccess) {
                   PR_fprintf(PR_STDERR, "Create Certificate Request: Failed\n");
                   goto cleanup;
               }
               break;
           case ADD_CERT_TO_DB:
               ValidateAddCertToDBCommand(progName, dbdir, nickNameStr, trustStr,
                                          certFileName, certReqFileName,
                                          issuerNameStr, serialNumberStr, selfsign);
               /* Add cert to database */
               rv = AddCertificateToDB(slot, &pwdata, certReqFileName, certFileName,
                                       issuerNameStr, certHandle, nickNameStr,
                                       trustStr, serialNumber, selfsign, ascii);
               if (rv != SECSuccess) {
                   PR_fprintf(PR_STDERR, "Add Certificate to DB: Failed\n");
                    goto cleanup;
               }
               break;
           case SAVE_CERT_TO_HEADER:
               ValidateSaveCertToHeaderCommand(progName, dbdir, nickNameStr, headerFileName);
               /* Save cert to the header file */
               rv = AddCertificateToHeader(slot, &pwdata, headerFileName, certHandle, nickNameStr, sigVerify);
               if (rv != SECSuccess) {
                   PR_fprintf(PR_STDERR, "Saving Certificate to header: Failed\n");
                   goto cleanup;
               }
               break;
           case ENCRYPT:
               ValidateEncryptCommand(progName, dbdir, nickNameStr, headerFileName, inFileName, encryptedFileName);
               /* Encrypt with public key from cert in header file */
               rv = FindKeyAndEncrypt(slot, &pwdata, headerFileName, encryptedFileName, inFileName);
               if (rv != SECSuccess) {
                   PR_fprintf(PR_STDERR, "Find public key and Encrypt : Failed\n");
                   goto cleanup;
               }
               break;
           case SIGN:
               ValidateSignCommand(progName, dbdir, nickNameStr, headerFileName, inFileName);
               /* Sign with private key */
               rv = FindKeyAndSign(slot, certHandle, &pwdata, nickNameStr, headerFileName, inFileName);
               if (rv != SECSuccess) {
                   PR_fprintf(PR_STDERR, "Find private key and sign : Failed\n");
                   goto cleanup;
               }
               break;
           case DECRYPT:
               ValidateDecryptCommand(progName, dbdir, headerFileName, encryptedFileName, outFileName);
               /* Decrypt with the matching private key */
               rv = FindKeyAndDecrypt(slot, &pwdata, headerFileName, encryptedFileName, outFileName);
               if (rv != SECSuccess) {
                   PR_fprintf(PR_STDERR, "Find private key and Decrypt : Failed\n");
               }
               break;
           case VERIFY:
               ValidateVerifyCommand(progName, dbdir, headerFileName, inFileName);
               /* Verify with the matching public key */
               rv = FindKeyAndVerify(slot, certHandle, &pwdata, headerFileName, inFileName);
               if (rv != SECSuccess) {
                   PR_fprintf(PR_STDERR, "Find public key and verify signature : Failed\n");
                   goto cleanup;
               }
           }
       cleanup:
           if (slot) {
               PK11_FreeSlot(slot);
           }
           if (initialized) {
               SECStatus rvShutdown = NSS_Shutdown();
               if (rvShutdown != SECSuccess) {
                   PR_fprintf(PR_STDERR, "Failed : NSS_Shutdown() - %s",
                              PORT_ErrorToString(rvShutdown));
                   rv = SECFailure;
               }
               PR_Cleanup();
           }
           return rv;
       }