Testing rats-tls remote attestation across TEE types
====

# Introduction

The [rats-tls](https://github.com/inclavare-containers/rats-tls) library contains support for both TDX ECDSA and SGX ECDSA remote attestation. This document demonstrates how two processes, located in a TD VM and an SGX enclave respectively, can verify each other in a trusted manner using the rats-tls library and Interoperable RA-TLS certificate.

We will use the sample programs "rats-tls-server" and "rats-tls-client" provided in the rats-tls library to conduct our demonstration. If you need to learn how to use rats-tls, please refer to our sample code [here](https://github.com/inclavare-containers/rats-tls/tree/master/samples).

# Build rats-tls

Firstly, we need to compile rats-tls wth all sample programs. You will need to complete this step separately in the TD VM and the SGX enclave.

## Compile on an SGX-enabled server

1. Firstly, please refer to [this guide](https://github.com/inclavare-containers/rats-tls#build-requirements) to install the packages required to build rats-tls.
   
    Alternatively, we recommend using the Docker image we provide to set up your compilation environment.
    ```sh
    docker run -it --privileged --network host \
        -v /dev/sgx_enclave:/dev/sgx/enclave \
        -v /dev/sgx_provision:/dev/sgx/provision \
        -v /var/run/aesmd:/var/run/aesmd \
        runetest/compilation-testing:ubuntu20.04
    ```
    Next, you can run the subsequent steps in that container.

2. Install and config Quote Provider Library

    Install `libsgx-dcap-default-qpl-dev` package, which is required to get collaterals and verify the quote.
    ```sh
    apt install -y libsgx-dcap-default-qpl-dev
    ```

    Open `/etc/sgx_default_qcnl.conf` and edit `PCCS_URL` to be the URL of your PCCS service.

3. Get source code of rats-tls

    ```sh
    git clone https://github.com/inclavare-containers/rats-tls
    ```

4. Compile rats-tls and the sample programs.

    > Note: Due to the limitations of occlum, we cannot yet verify tdx quote in occlum, so in this article we will run rats-tls based on sgxsdk.

    ```sh
    cd rats-tls

    # SGX mode
    cmake -DRATS_TLS_BUILD_MODE="sgx" -DBUILD_SAMPLES=on -H. -Bbuild
    make -C build install
    ```
    
## Compile on an TD VM

1. Firstly, you can install your software packages in the same way as you did in SGX.

    Alternatively, we recommend using a Docker image to initialize your compilation environment.

    ```sh
    docker run -it --privileged --network host \
        -v /dev/tdx-guest:/dev/tdx-guest \
        -v /var/run/aesmd:/var/run/aesmd \
        runetest/compilation-testing:ubuntu20.04
    ```

    > Note: If rats-tls complaints about file not found, please make sure you have upgraded to the latest SGX libraries using `apt upgrade`.

2. Install and config Quote Provider Library

    Same steps as in SGX-enabled server.

3. (Optional) Compile and install newer openssl

    Previous versions of openssl had a bug: when the client side certificate exceeded a certain size, the client failed to receive data from the server side. This will cause our case 3 to fail. We recommend to upgrade your openssl version to at least 1.1.1q to avoid this problem.

    Compile and install openssl.
    ```sh
    pushd /tmp/
    wget https://www.openssl.org/source/openssl-1.1.1q.tar.gz
    tar -xzvf openssl-1.1.1q.tar.gz
    cd openssl-1.1.1q
    ./config --prefix=/opt/openssl-1.1.1q
    make -j && make install
    popd /tmp/
    ```
    
    Add to library search path so that dynamic loader can find it.
    ```sh
    echo '/opt/openssl-1.1.1q/lib' > /etc/ld.so.conf.d/openssl-1.1.1q.conf
    ldconfig -v
    ```

    Now run `openssl version` and you will see that `1.1.1q` is used.    
    ```txt
    OpenSSL 1.1.1f  31 Mar 2020 (Library: OpenSSL 1.1.1q  5 Jul 2022)
    ```

4. Get source code of rats-tls

    ```sh
    git clone https://github.com/inclavare-containers/rats-tls
    ```

5. Compile rats-tls and the sample programs.

    ```sh
    cd rats-tls

    # TDX mode
    cmake -DRATS_TLS_BUILD_MODE="tdx" -DBUILD_SAMPLES=on -H. -Bbuild
    make -C build install
    ```

# Test cases

## Case 1: SGX enclave as the attester and TD VM as the verifier

1. Run `rats-tls-server` in the SGX enclave with SGX ECDSA attester.

    ```sh
    cd /usr/share/rats-tls/samples
    
    ./rats-tls-server --log-level debug --ip 0.0.0.0 --port 1234 --attester sgx_ecdsa --verifier nullverifier --crypto openssl --tls openssl --endorsements
    ```

    Now, the rats-tls-server is listening on port 1234.

2. Run `rats-tls-client` in the TD VM with SGX ECDSA verifier.

    ```sh
    cd /usr/share/rats-tls/samples

    ./rats-tls-client --log-level debug --ip <sgx-server-ip> --port 1234 --attester nullattester --verifier sgx_ecdsa --crypto openssl --tls openssl
    ```
    
    Where `<sgx-server-ip>` is the IP address of the rats-tls-server.

## Case 2: TD VM as the attester and SGX enclave as the verifier

1. Run `rats-tls-server` in the TD VM with TDX ECDSA attester.

    ```sh
    cd /usr/share/rats-tls/samples

    ./rats-tls-server --log-level debug --ip 0.0.0.0 --port 1234 --attester tdx_ecdsa --verifier nullverifier --crypto openssl --tls openssl --endorsements
    ```
    Now, the rats-tls-server is listening on port 1234.

2. Run `rats-tls-client` in the SGX enclave with TDX ECDSA verifier.
    ```sh
    cd /usr/share/rats-tls/samples

    ./rats-tls-client --log-level debug --ip <td-server-ip> --port 1234 --attester nullattester --verifier tdx_ecdsa --crypto openssl --tls openssl
    ```

    Where `<td-server-ip>` is the IP address of the rats-tls-server.

## Case 3: TD VM and SGX enclave as both attester and verifier (mutual attestation).

This case builds upon Case 1, adding verification of the rats-tls-client, which is based on OpenSSL client cert.

1. Run `rats-tls-server` in the SGX enclave and enable both the SGX ECDSA attester and the TDX ECDSA verifier.

    ```sh
    cd /usr/share/rats-tls/samples

    ./rats-tls-server --log-level debug --ip 0.0.0.0 --port 1234 --attester sgx_ecdsa --verifier tdx_ecdsa --crypto openssl --tls openssl --endorsements --mutual
    ```

    Now, the rats-tls-server is listening on port 1234.

2. Run `rats-tls-client` in the TD VM and enable both the TDX ECDSA attester and the SGX ECDSA verifier.

    ```sh
    cd /usr/share/rats-tls/samples

    ./rats-tls-client --log-level debug --ip <sgx-server-ip> --port 1234 --attester tdx_ecdsa --verifier sgx_ecdsa --crypto openssl --tls openssl --endorsements --mutual
    ```
    
    Where `<sgx-server-ip>` is the IP address of the rats-tls-server.

