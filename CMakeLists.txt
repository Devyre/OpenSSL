cmake_minimum_required(VERSION 3.18)
include(CheckTypeSize)

macro(add_openssl_source VAR DIR)
    foreach(name ${ARGN})
        list(APPEND ${VAR} ${openssl_SOURCE_DIR}/crypto/${DIR}/${name})
    endforeach()
endmacro()
function(openssl_common_settings)
    cmake_parse_arguments(
            ocs
            ""
            "TARGET"
            ""
            ${ARGN}
    )

    target_compile_definitions(
            ${ocs_TARGET}
            PRIVATE
            OPENSSLDIR="${OPENSSL_OPENSSLDIR}"
            ENGINESDIR="${OPENSSL_ENGINESDIR}"
    )
    target_compile_definitions(
            ${ocs_TARGET}
            PUBLIC
            OPENSSL_NO_ASM
            OPENSSL_NO_STATIC_ENGINE
    )

    if (APPLE)
        target_compile_definitions(
                ${ocs_TARGET}
                PRIVATE
                OPENSSL_SYSNAME_MACOSX
        )
    endif()
    if (WIN32 AND NOT CYGWIN)
        target_compile_definitions(
                ${ocs_TARGET}
                PRIVATE
                OPENSSL_SYSNAME_WIN32
                WIN32_LEAN_AND_MEAN
        )

        if (OPENSSL_SHARED_LIB)
            target_compile_definitions(
                    ${ocs_TARGET}
                    PRIVATE
                    _WINDLL
            )

            set_target_properties(
                    ${ocs_TARGET}
                    PROPERTIES
                    WINDOWS_EXPORT_ALL_SYMBOLS ON
            )
        endif()
    endif()

    if (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC" OR CMAKE_CXX_COMPILER_FRONTEND_VARIANT STREQUAL "MSVC")
        target_compile_options(${ocs_TARGET} PRIVATE /W0)
    else()
        target_compile_options(${ocs_TARGET} PRIVATE -w)
    endif()
endfunction()

option(OPENSSL_SHARED_LIB "Build OpenSSL as shared libraries" OFF)

if (NOT DEFINED OCTANE_EXTERNAL_DIR)
    set(OCTANE_EXTERNAL_DIR $ENV{OCTANE_EXTERNAL_DIR})

    if (NOT DEFINED OCTANE_EXTERNAL_DIR)
        set(OCTANE_EXTERNAL_DIR "${CMAKE_CURRENT_BINARY_DIR}/external")
    endif()
endif()

if (NOT DEFINED OCTANE_OPENSSL_VERSION)
    set(OCTANE_OPENSSL_VERSION $ENV{OCTANE_OPENSSL_VERSION})

    if (NOT DEFINED OCTANE_OPENSSL_VERSION)
        set(OCTANE_OPENSSL_VERSION "1.1.1k")
    endif()
endif()

if (NOT DEFINED OCTANE_OPENSSL_URL)
    set(OCTANE_OPENSSL_URL $ENV{OCTANE_OPENSSL_URL})

    if (NOT DEFINED OCTANE_OPENSSL_URL)
        set(OCTANE_OPENSSL_URL "https://www.openssl.org/source/openssl-${OCTANE_OPENSSL_VERSION}.tar.gz")
    endif()
endif()

message(STATUS "OpenSSL target directory: ${OCTANE_EXTERNAL_DIR}/lib/openssl_${OCTANE_OPENSSL_VERSION}")
message(STATUS "Configuring OpenSSL ${OCTANE_OPENSSL_VERSION}")

file(LOCK "${OCTANE_EXTERNAL_DIR}/lib/openssl.lock")
set(openssl_SOURCE_DIR "${OCTANE_EXTERNAL_DIR}/lib/openssl-${OCTANE_OPENSSL_VERSION}")

if (NOT EXISTS "${OCTANE_EXTERNAL_DIR}/lib/openssl-${OCTANE_OPENSSL_VERSION}.tar.gz")
    message(STATUS "Downloading OpenSSL ${OCTANE_OPENSSL_VERSION} from ${OCTANE_OPENSSL_URL}...")
    file(DOWNLOAD "${OCTANE_OPENSSL_URL}" "${OCTANE_EXTERNAL_DIR}/lib/openssl-${OCTANE_OPENSSL_VERSION}.tar.gz")
endif()

if (NOT EXISTS "${OCTANE_EXTERNAL_DIR}/lib/openssl-${OCTANE_OPENSSL_VERSION}")
    message(STATUS "Extracting openssl-${OCTANE_OPENSSL_VERSION}.tar.gz...")
    file(ARCHIVE_EXTRACT INPUT "${OCTANE_EXTERNAL_DIR}/lib/openssl-${OCTANE_OPENSSL_VERSION}.tar.gz" DESTINATION "${OCTANE_EXTERNAL_DIR}/lib")
endif()

file(LOCK "${OCTANE_EXTERNAL_DIR}/lib/openssl.lock" RELEASE)

if (WIN32 AND NOT CYGWIN)
    set(OPENSSL_EXPORT_VAR_AS_FUNCTION 1)
endif()

if (WIN32)
    set(OPENSSL_OPENSSLDIR "C:/ssl")
    set(OPENSSL_ENGINESDIR "C:/engines-1.1")
else()
    set(OPENSSL_OPENSSLDIR "/usr/local/ssl")
    set(OPENSSL_ENGINESDIR "/usr/local/engines-1.1")
endif()

check_type_size("long" LONG_INT)
check_type_size("long long" LONG_LONG_INT)
check_type_size("int" INT)

if(HAVE_LONG_INT AND "${LONG_INT}" STREQUAL "8")
    set(SIXTY_FOUR_BIT_LONG ON)
elseif(HAVE_LONG_LONG_INT AND "${LONG_LONG_INT}" STREQUAL "8")
    set(SIXTY_FOUR_BIT ON)
else()
    set(THIRTY_TWO_BIT ON)
endif()

configure_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/opensslconf.h.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/openssl/opensslconf.h
)

string(TIMESTAMP BUILDINF_DATE "%Y-%m-%d %H:%M:%S UTC" UTC)
configure_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/buildinf.h.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/buildinf.h
)

check_include_file(fcntl.h CHK_DLFCN)
if (CHK_DLFCN)
    set(HAVE_DLFCN_H ON)
else()
    set(HAVE_DLFCN_H OFF)
endif()

if (APPLE)
    set(DSO_DLFCN ON)
    set(DSO_EXTENSION ".dylib")
elseif (WIN32 AND NOT CYGWIN)
    set(DSO_WIN32 ON)
    set(DSO_EXTENSION ".dll")
elseif (CYGWIN)
    set(DSO_DLFCN ON)
    set(DSO_EXTENSION ".dll")
else()
    if (CHK_DLFCN)
        set(DSO_DLFCN ON)
    else()
        set(DSO_NONE ON)
    endif()
    set(DSO_EXTENSION ".so")
endif()

configure_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/bn_conf.h.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/crypto/bn_conf.h
)
configure_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/dso_conf.h.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/crypto/dso_conf.h
)

set(OPENSSL_CRYPTO_SRC
        ${openssl_SOURCE_DIR}/crypto/cpt_err.c
        ${openssl_SOURCE_DIR}/crypto/cryptlib.c
        ${openssl_SOURCE_DIR}/crypto/ctype.c
        ${openssl_SOURCE_DIR}/crypto/cversion.c
        ${openssl_SOURCE_DIR}/crypto/ebcdic.c
        ${openssl_SOURCE_DIR}/crypto/ex_data.c
        ${openssl_SOURCE_DIR}/crypto/init.c
        ${openssl_SOURCE_DIR}/crypto/mem.c
        ${openssl_SOURCE_DIR}/crypto/mem_clr.c
        ${openssl_SOURCE_DIR}/crypto/mem_dbg.c
        ${openssl_SOURCE_DIR}/crypto/mem_sec.c
        ${openssl_SOURCE_DIR}/crypto/o_dir.c
        ${openssl_SOURCE_DIR}/crypto/o_fips.c
        ${openssl_SOURCE_DIR}/crypto/o_fopen.c
        ${openssl_SOURCE_DIR}/crypto/o_init.c
        ${openssl_SOURCE_DIR}/crypto/o_str.c
        ${openssl_SOURCE_DIR}/crypto/o_time.c
        ${openssl_SOURCE_DIR}/crypto/uid.c
        ${openssl_SOURCE_DIR}/crypto/getenv.c
        )
add_openssl_source(OPENSSL_CRYPTO_SRC aes aes_cbc.c aes_cfb.c aes_core.c aes_ecb.c aes_ige.c aes_misc.c aes_ofb.c aes_wrap.c)
add_openssl_source(OPENSSL_CRYPTO_SRC aria aria.c)
add_openssl_source(OPENSSL_CRYPTO_SRC asn1 a_bitstr.c a_d2i_fp.c a_digest.c a_dup.c a_gentm.c a_i2d_fp.c a_int.c
        a_mbstr.c a_object.c a_octet.c a_print.c a_sign.c a_strex.c a_strnid.c a_time.c a_type.c
        a_utctm.c a_utf8.c a_verify.c ameth_lib.c asn1_err.c asn1_gen.c asn1_item_list.c asn1_lib.c asn1_par.c
        asn_mime.c asn_moid.c asn_mstbl.c asn_pack.c bio_asn1.c bio_ndef.c d2i_pr.c d2i_pu.c
        evp_asn1.c f_int.c f_string.c i2d_pr.c i2d_pu.c n_pkey.c nsseq.c p5_pbe.c p5_pbev2.c
        p5_scrypt.c p8_pkey.c t_bitst.c t_pkey.c t_spki.c tasn_dec.c tasn_enc.c tasn_fre.c
        tasn_new.c tasn_prn.c tasn_scn.c tasn_typ.c tasn_utl.c x_algor.c x_bignum.c x_info.c
        x_int64.c x_long.c x_pkey.c x_sig.c x_spki.c x_val.c)
add_openssl_source(OPENSSL_CRYPTO_SRC async async async.c async_err.c async_wait.c
        arch/async_null.c arch/async_posix.c arch/async_win.c)
add_openssl_source(OPENSSL_CRYPTO_SRC bf bf_cfb64.c bf_ecb.c bf_enc.c bf_ofb64.c bf_skey.c)
add_openssl_source(OPENSSL_CRYPTO_SRC bio b_addr.c b_dump.c b_print.c b_sock.c b_sock2.c bf_buff.c bf_lbuf.c
        bf_nbio.c bf_null.c bio_cb.c bio_err.c bio_lib.c bio_meth.c bss_acpt.c bss_bio.c
        bss_conn.c bss_dgram.c bss_fd.c bss_file.c bss_log.c bss_mem.c bss_null.c bss_sock.c)
add_openssl_source(OPENSSL_CRYPTO_SRC blake2 blake2b.c blake2s.c m_blake2b.c m_blake2s.c)
add_openssl_source(OPENSSL_CRYPTO_SRC bn bn_add.c bn_asm.c bn_blind.c bn_const.c bn_ctx.c bn_depr.c bn_dh.c
        bn_div.c bn_err.c bn_exp.c bn_exp2.c bn_gcd.c bn_gf2m.c bn_intern.c bn_kron.c bn_lib.c
        bn_mod.c bn_mont.c bn_mpi.c bn_mul.c bn_nist.c bn_prime.c bn_print.c bn_rand.c bn_recp.c
        bn_shift.c bn_sqr.c bn_sqrt.c bn_srp.c bn_word.c bn_x931p.c rsaz_exp.c)
add_openssl_source(OPENSSL_CRYPTO_SRC buffer buf_err.c buffer.c)
add_openssl_source(OPENSSL_CRYPTO_SRC camellia camellia.c cmll_cbc.c cmll_cfb.c cmll_ctr.c cmll_ecb.c
        cmll_misc.c cmll_ofb.c)
add_openssl_source(OPENSSL_CRYPTO_SRC cast c_cfb64.c c_ecb.c c_enc.c c_ofb64.c c_skey.c)
add_openssl_source(OPENSSL_CRYPTO_SRC chacha chacha_enc.c)
add_openssl_source(OPENSSL_CRYPTO_SRC cmac cm_ameth.c cm_pmeth.c cmac.c)
add_openssl_source(OPENSSL_CRYPTO_SRC cms cms_asn1.c cms_att.c cms_cd.c cms_dd.c cms_enc.c cms_env.c cms_err.c
        cms_ess.c cms_io.c cms_kari.c cms_lib.c cms_pwri.c cms_sd.c cms_smime.c)
add_openssl_source(OPENSSL_CRYPTO_SRC comp c_zlib.c comp_err.c comp_lib.c)
add_openssl_source(OPENSSL_CRYPTO_SRC conf conf_api.c conf_def.c conf_err.c conf_lib.c conf_mall.c conf_mod.c
        conf_sap.c conf_ssl.c)
add_openssl_source(OPENSSL_CRYPTO_SRC ct ct_b64.c ct_err.c ct_log.c ct_oct.c ct_policy.c ct_prn.c ct_sct.c
        ct_sct_ctx.c ct_vfy.c ct_x509v3.c)
add_openssl_source(OPENSSL_CRYPTO_SRC des cbc_cksm.c cbc_enc.c cfb64ede.c cfb64enc.c cfb_enc.c des_enc.c
        ecb3_enc.c ecb_enc.c fcrypt.c fcrypt_b.c ofb64ede.c ofb64enc.c ofb_enc.c pcbc_enc.c
        qud_cksm.c rand_key.c set_key.c str2key.c xcbc_enc.c)
add_openssl_source(OPENSSL_CRYPTO_SRC dh dh_ameth.c dh_asn1.c dh_check.c dh_depr.c dh_err.c dh_gen.c dh_kdf.c
        dh_key.c dh_lib.c dh_meth.c dh_pmeth.c dh_prn.c dh_rfc5114.c dh_rfc7919.c)
add_openssl_source(OPENSSL_CRYPTO_SRC dsa dsa_ameth.c dsa_asn1.c dsa_depr.c dsa_err.c dsa_gen.c dsa_key.c
        dsa_lib.c dsa_meth.c dsa_ossl.c dsa_pmeth.c dsa_prn.c dsa_sign.c dsa_vrf.c)
add_openssl_source(OPENSSL_CRYPTO_SRC dso dso_dl.c dso_dlfcn.c dso_err.c dso_lib.c dso_openssl.c dso_vms.c
        dso_win32.c)
add_openssl_source(OPENSSL_CRYPTO_SRC ec curve25519.c ec2_oct.c ec2_smpl.c ec_ameth.c ec_asn1.c
        ec_check.c ec_curve.c ec_cvt.c ec_err.c ec_key.c ec_kmeth.c ec_lib.c ec_mult.c
        ec_oct.c ec_pmeth.c ec_print.c ecdh_kdf.c ecdh_ossl.c ecdsa_ossl.c ecdsa_sign.c
        ecdsa_vrf.c eck_prn.c ecp_mont.c ecp_nist.c ecp_nistp224.c ecp_nistp256.c ecp_nistp521.c
        ecp_nistputil.c ecp_oct.c ecp_smpl.c ecx_meth.c
        curve448/curve448.c curve448/curve448_tables.c curve448/eddsa.c curve448/f_generic.c
        curve448/scalar.c curve448/arch_32/f_impl.c)
add_openssl_source(OPENSSL_CRYPTO_SRC engine eng_all.c eng_cnf.c eng_ctrl.c eng_dyn.c eng_err.c
        eng_fat.c eng_init.c eng_lib.c eng_list.c eng_openssl.c eng_pkey.c eng_rdrand.c
        eng_table.c tb_asnmth.c tb_cipher.c tb_dh.c tb_digest.c tb_dsa.c tb_eckey.c tb_pkmeth.c
        tb_rand.c tb_rsa.c)
add_openssl_source(OPENSSL_CRYPTO_SRC err err.c err_all.c err_prn.c)
add_openssl_source(OPENSSL_CRYPTO_SRC evp bio_b64.c bio_enc.c bio_md.c bio_ok.c c_allc.c c_alld.c cmeth_lib.c
        digest.c e_aes.c e_aes_cbc_hmac_sha1.c e_aes_cbc_hmac_sha256.c e_aria.c e_bf.c e_camellia.c
        e_cast.c e_chacha20_poly1305.c e_des.c e_des3.c e_idea.c e_null.c e_old.c e_rc2.c
        e_rc4.c e_rc4_hmac_md5.c e_rc5.c e_sm4.c e_seed.c e_xcbc_d.c encode.c evp_cnf.c evp_enc.c
        evp_err.c evp_key.c evp_lib.c evp_pbe.c evp_pkey.c m_md2.c m_md4.c m_md5.c m_md5_sha1.c
        m_sha3.c m_mdc2.c m_null.c m_ripemd.c m_sha1.c m_sigver.c m_wp.c names.c p5_crpt.c p5_crpt2.c
        p_dec.c p_enc.c p_lib.c p_open.c p_seal.c p_sign.c p_verify.c pbe_scrypt.c
        pmeth_fn.c pmeth_gn.c pmeth_lib.c)
add_openssl_source(OPENSSL_CRYPTO_SRC hmac hm_ameth.c hm_pmeth.c hmac.c)
add_openssl_source(OPENSSL_CRYPTO_SRC idea i_cbc.c i_cfb64.c i_ecb.c i_ofb64.c i_skey.c)
add_openssl_source(OPENSSL_CRYPTO_SRC kdf hkdf.c kdf_err.c scrypt.c tls1_prf.c)
add_openssl_source(OPENSSL_CRYPTO_SRC lhash lh_stats.c lhash.c)
add_openssl_source(OPENSSL_CRYPTO_SRC md4 md4_dgst.c md4_one.c)
add_openssl_source(OPENSSL_CRYPTO_SRC md5 md5_dgst.c md5_one.c)
add_openssl_source(OPENSSL_CRYPTO_SRC mdc2 mdc2_one.c mdc2dgst.c)
add_openssl_source(OPENSSL_CRYPTO_SRC modes cbc128.c ccm128.c cfb128.c ctr128.c cts128.c gcm128.c ocb128.c
        ofb128.c wrap128.c xts128.c)
add_openssl_source(OPENSSL_CRYPTO_SRC objects o_names.c obj_dat.c obj_err.c obj_lib.c obj_xref.c)
add_openssl_source(OPENSSL_CRYPTO_SRC ocsp ocsp_asn.c ocsp_cl.c ocsp_err.c ocsp_ext.c ocsp_ht.c ocsp_lib.c
        ocsp_prn.c ocsp_srv.c ocsp_vfy.c v3_ocsp.c)
add_openssl_source(OPENSSL_CRYPTO_SRC pem pem_all.c pem_err.c pem_info.c pem_lib.c pem_oth.c pem_pk8.c pem_pkey.c
        pem_sign.c pem_x509.c pem_xaux.c pvkfmt.c)
add_openssl_source(OPENSSL_CRYPTO_SRC pkcs12 p12_add.c p12_asn.c p12_attr.c p12_crpt.c p12_crt.c p12_decr.c
        p12_init.c p12_key.c p12_kiss.c p12_mutl.c p12_npas.c p12_p8d.c p12_p8e.c p12_sbag.c
        p12_utl.c pk12err.c)
add_openssl_source(OPENSSL_CRYPTO_SRC pkcs7 bio_pk7.c pk7_asn1.c pk7_attr.c pk7_doit.c pk7_lib.c pk7_mime.c
        pk7_smime.c pkcs7err.c)
add_openssl_source(OPENSSL_CRYPTO_SRC poly1305 poly1305.c poly1305_ameth.c poly1305_pmeth.c)
add_openssl_source(OPENSSL_CRYPTO_SRC rand drbg_ctr.c drbg_lib.c rand_egd.c rand_err.c rand_lib.c rand_unix.c rand_vms.c
        rand_win.c randfile.c)
add_openssl_source(OPENSSL_CRYPTO_SRC rc2 rc2_cbc.c rc2_ecb.c rc2_skey.c rc2cfb64.c rc2ofb64.c)
add_openssl_source(OPENSSL_CRYPTO_SRC rc4 rc4_enc.c rc4_skey.c)
add_openssl_source(OPENSSL_CRYPTO_SRC rc5 rc5_ecb.c rc5_enc.c rc5_skey.c rc5cfb64.c rc5ofb64.c)
add_openssl_source(OPENSSL_CRYPTO_SRC ripemd rmd_dgst.c rmd_one.c)
add_openssl_source(OPENSSL_CRYPTO_SRC rsa rsa_ameth.c rsa_asn1.c rsa_chk.c rsa_crpt.c rsa_depr.c rsa_err.c
        rsa_gen.c rsa_lib.c rsa_meth.c rsa_mp.c rsa_none.c rsa_oaep.c rsa_ossl.c rsa_pk1.c
        rsa_pmeth.c rsa_prn.c rsa_pss.c rsa_saos.c rsa_sign.c rsa_ssl.c rsa_x931.c rsa_x931g.c)
add_openssl_source(OPENSSL_CRYPTO_SRC seed seed.c seed_cbc.c seed_cfb.c seed_ecb.c seed_ofb.c)
add_openssl_source(OPENSSL_CRYPTO_SRC sha keccak1600.c sha1_one.c sha1dgst.c sha256.c sha512.c)
add_openssl_source(OPENSSL_CRYPTO_SRC siphash siphash.c siphash_ameth.c siphash_pmeth.c)
add_openssl_source(OPENSSL_CRYPTO_SRC sm2 sm2_crypt.c sm2_err.c sm2_pmeth.c sm2_sign.c)
add_openssl_source(OPENSSL_CRYPTO_SRC sm3 m_sm3.c sm3.c)
add_openssl_source(OPENSSL_CRYPTO_SRC sm4 sm4.c)
add_openssl_source(OPENSSL_CRYPTO_SRC srp srp_lib.c srp_vfy.c)
add_openssl_source(OPENSSL_CRYPTO_SRC stack stack.c)
add_openssl_source(OPENSSL_CRYPTO_SRC store loader_file.c store_err.c store_init.c store_lib.c store_register.c store_strings.c)
add_openssl_source(OPENSSL_CRYPTO_SRC ts ts_asn1.c ts_conf.c ts_err.c ts_lib.c ts_req_print.c ts_req_utils.c
        ts_rsp_print.c ts_rsp_sign.c ts_rsp_utils.c ts_rsp_verify.c ts_verify_ctx.c)
add_openssl_source(OPENSSL_CRYPTO_SRC txt_db txt_db.c)
add_openssl_source(OPENSSL_CRYPTO_SRC ui ui_err.c ui_lib.c ui_null.c ui_openssl.c ui_util.c)
add_openssl_source(OPENSSL_CRYPTO_SRC whrlpool wp_block.c wp_dgst.c)
add_openssl_source(OPENSSL_CRYPTO_SRC x509 by_dir.c by_file.c t_crl.c t_req.c t_x509.c x509_att.c x509_cmp.c
        x509_d2.c x509_def.c x509_err.c x509_ext.c x509_lu.c x509_meth.c x509_obj.c x509_r2x.c
        x509_req.c x509_set.c x509_trs.c x509_txt.c x509_v3.c x509_vfy.c x509_vpm.c x509cset.c
        x509name.c x509rset.c x509spki.c x509type.c x_all.c x_attrib.c x_crl.c x_exten.c x_name.c
        x_pubkey.c x_req.c x_x509.c x_x509a.c)
add_openssl_source(OPENSSL_CRYPTO_SRC x509v3 pcy_cache.c pcy_data.c pcy_lib.c pcy_map.c pcy_node.c pcy_tree.c
        v3_addr.c v3_admis.c v3_akey.c v3_akeya.c v3_alt.c v3_asid.c v3_bcons.c v3_bitst.c v3_conf.c v3_cpols.c
        v3_crld.c v3_enum.c v3_extku.c v3_genn.c v3_ia5.c v3_info.c v3_int.c v3_lib.c v3_ncons.c
        v3_pci.c v3_pcia.c v3_pcons.c v3_pku.c v3_pmaps.c v3_prn.c v3_purp.c v3_skey.c v3_sxnet.c
        v3_tlsf.c v3_utl.c v3err.c)

find_package(Threads)
if (NOT Threads_FOUND)
    add_openssl_source(OPENSSL_CRYPTO_SRC . threads_none.c)
elseif (WIN32)
    add_openssl_source(OPENSSL_CRYPTO_SRC . threads_win.c)
else()
    add_openssl_source(OPENSSL_CRYPTO_SRC . threads_pthread.c)
endif()

if (OPENSSL_SHARED_LIB)
    set(OPENSSL_LIB_TYPE SHARED)
else()
    set(OPENSSL_LIB_TYPE STATIC)
endif()

add_library(
        openssl_crypto
        ${OPENSSL_LIB_TYPE}
        ${OPENSSL_CRYPTO_SRC}
)
target_include_directories(
        openssl_crypto
        PUBLIC
        ${openssl_SOURCE_DIR}/crypto/modes
        ${openssl_SOURCE_DIR}/crypto/ec/curve448
        ${openssl_SOURCE_DIR}/crypto/ec/curve448/arch_32
        ${CMAKE_CURRENT_BINARY_DIR}
        ${openssl_SOURCE_DIR}/include
        ${openssl_SOURCE_DIR}
)
if (OPENSSL_SHARED_LIB)
    target_compile_definitions(
            openssl_crypto
            PRIVATE
            OPENSSL_BUILD_SHLIBCRYPTO
    )
endif()
if (WIN32 AND NOT CYGWIN)
    target_link_libraries(openssl_crypto PUBLIC ws2_32 crypt32)
else()
    if (Threads_FOUND)
        target_link_libraries(openssl_crypto PUBLIC ${CMAKE_THREAD_LIBS_INIT})
    endif()
    if (DSO_DLFCN AND HAVE_DLFCN_H)
        target_link_libraries(openssl_crypto PUBLIC dl)
    endif()
endif()

openssl_common_settings(TARGET openssl_crypto)

file(GLOB_RECURSE OPENSSL_SSL_SRC ${openssl_SOURCE_DIR}/ssl/*.c)

add_library(
        openssl_ssl
        ${OPENSSL_LIB_TYPE}
        ${OPENSSL_SSL_SRC}
)
target_include_directories(
        openssl_ssl
        PUBLIC
        ${openssl_SOURCE_DIR}/include
)
target_link_libraries(
        openssl_ssl
        PUBLIC
        openssl_crypto
)
if (OPENSSL_SHARED_LIB)
    target_compile_definitions(
            openssl_ssl
            PRIVATE
            OPENSSL_BUILD_SHLIBSSL
    )
endif()

openssl_common_settings(TARGET openssl_ssl)

add_library(OpenSSL INTERFACE)
target_link_libraries(OpenSSL
        INTERFACE openssl_crypto
        INTERFACE openssl_ssl
)
