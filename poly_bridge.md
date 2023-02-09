# Introduction

|                   | External                                                                 | Local                                                                      | Native                                                |
|-------------------|--------------------------------------------------------------------------|----------------------------------------------------------------------------|-------------------------------------------------------|
| Mechanism         | External actors attest message correctness.                              | Messages are verified by the parties who they affect.                      | Messages are verified directly by chain validator set |
| Trust assumptions | Fully assumes honesty of verifier set quorum.                            | Assumes liveness of all parties within a certain period to verify message. | Optimistic(Fraud Proof) or ZKP(Validity Proofs)       |
| Tradeoffs         | Heterogeneous blockchains are supported and liquidity can be aggregated. | Doesn't really scale beyond 2 parties.                                     | Heterogeneous blockchains are not supported.          |
| Examples          | LayerZero, Celer, Multichain, Poly                                       | Connext v0, Hop                                                            | Optimism, Arbitrum, ZkSync, Starkware                 |

Among them, Poly bridge belongs to the **External** category.

# Protocol

1. The asset is locked on source chain.
2. After the tx is **confirmed** on source chain, it is relayed to a relay blockchain run by trusted verifier set.
3. After the verifier set quorum confirms with a multi-signature, it is relayed to the target chain. The smart contract on the target chain verifies the fact and then transfers the target asset to the target address.

# Implementation

Let's take the crosschain process between two EVM blokchains as an example.

1. The asset is locked on source chain via a [`lock`](https://github.com/polynetwork/eth-contracts/blob/master/contracts/core/lock_proxy/LockProxy.sol#L64) function.
    ```solidity
    /* @notice                  This function is meant to be invoked by the user,
    *                           a certin amount teokens will be locked in the proxy contract the invoker/msg.sender immediately.
    *                           Then the same amount of tokens will be unloked from target chain proxy contract at the target chain with chainId later.
    *  @param fromAssetHash     The asset address in current chain, uniformly named as `fromAssetHash`
    *  @param toChainId         The target chain id
    *                           
    *  @param toAddress         The address in bytes format to receive same amount of tokens in target chain 
    *  @param amount            The amount of tokens to be crossed from ethereum to the chain with chainId
    */
    function lock(address fromAssetHash, uint64 toChainId, bytes memory toAddress, uint256 amount) public payable returns (bool) {
        require(amount != 0, "amount cannot be zero!");
        
        
        require(_transferToContract(fromAssetHash, amount), "transfer asset from fromAddress to lock_proxy contract  failed!");
        
        bytes memory toAssetHash = assetHashMap[fromAssetHash][toChainId];
        require(toAssetHash.length != 0, "empty illegal toAssetHash");

        TxArgs memory txArgs = TxArgs({
            toAssetHash: toAssetHash,
            toAddress: toAddress,
            amount: amount
        });
        bytes memory txData = _serializeTxArgs(txArgs);
        
        IEthCrossChainManagerProxy eccmp = IEthCrossChainManagerProxy(managerProxyContract);
        address eccmAddr = eccmp.getEthCrossChainManager();
        IEthCrossChainManager eccm = IEthCrossChainManager(eccmAddr);
        
        bytes memory toProxyHash = proxyHashMap[toChainId];
        require(toProxyHash.length != 0, "empty illegal toProxyHash");
        require(eccm.crossChain(toChainId, toProxyHash, "unlock", txData), "EthCrossChainManager crossChain executed error!");

        emit LockEvent(fromAssetHash, _msgSender(), toChainId, toAssetHash, toAddress, amount);
        
        return true;

    }
    ```
2. After the tx is **confirmed** on source chain, it is relayed to a relay blockchain run by trusted verifier set.
    1. There's a [`ImportExTransfer`](https://github.com/polynetwork/poly/blob/master/native/service/cross_chain_manager/entrance.go#L111) function on relay blockchain:
        ```golang
        func ImportExTransfer(native *native.NativeService) ([]byte, error) {
            params := new(scom.EntranceParam)
            if err := params.Deserialization(common.NewZeroCopySource(native.GetInput())); err != nil {
                return utils.BYTE_FALSE, fmt.Errorf("ImportExTransfer, contract params deserialize error: %v", err)
            }

            chainID := params.SourceChainID
            blacked, err := scom.CheckIfChainBlacked(native, chainID)
            if err != nil {
                return utils.BYTE_FALSE, fmt.Errorf("ImportExTransfer, CheckIfChainBlacked error: %v", err)
            }
            if blacked {
                return utils.BYTE_FALSE, fmt.Errorf("ImportExTransfer, source chain is blacked")
            }

            //check if chainid exist
            sideChain, err := side_chain_manager.GetSideChain(native, chainID)
            if err != nil {
                return utils.BYTE_FALSE, fmt.Errorf("ImportExTransfer, side_chain_manager.GetSideChain error: %v", err)
            }
            if sideChain == nil {
                return utils.BYTE_FALSE, fmt.Errorf("ImportExTransfer, side chain %d is not registered", chainID)
            }

            handler, err := GetChainHandler(sideChain.Router)
            if err != nil {
                return utils.BYTE_FALSE, err
            }
            err = utils.CheckRouterStartBlock(sideChain.Router, native.GetHeight())
            if err != nil {
                return utils.BYTE_FALSE, err
            }

            //1. verify tx
            txParam, err := handler.MakeDepositProposal(native)
            if err != nil {
                return utils.BYTE_FALSE, err
            }
            if txParam == nil && (sideChain.Router == utils.VOTE_ROUTER || sideChain.Router == utils.RIPPLE_ROUTER) {
                return utils.BYTE_TRUE, nil
            }

            //2. make target chain tx
            targetid := txParam.ToChainID
            blacked, err = scom.CheckIfChainBlacked(native, targetid)
            if err != nil {
                return utils.BYTE_FALSE, fmt.Errorf("ImportExTransfer, CheckIfChainBlacked error: %v", err)
            }
            if blacked {
                return utils.BYTE_FALSE, fmt.Errorf("ImportExTransfer, target chain is blacked")
            }

            //check if chainid exist
            sideChain, err = side_chain_manager.GetSideChain(native, targetid)
            if err != nil {
                return utils.BYTE_FALSE, fmt.Errorf("ImportExTransfer, side_chain_manager.GetSideChain error: %v", err)
            }
            if sideChain == nil {
                return utils.BYTE_FALSE, fmt.Errorf("ImportExTransfer, side chain %d is not registered", targetid)
            }
            if sideChain.Router == utils.BTC_ROUTER {
                err := btc.NewBTCHandler().MakeTransaction(native, txParam, chainID)
                if err != nil {
                    return utils.BYTE_FALSE, err
                }
                return utils.BYTE_TRUE, nil
            }

            if sideChain.Router == utils.RIPPLE_ROUTER {
                err := ripple.NewRippleHandler().MakeTransaction(native, txParam, chainID)
                if err != nil {
                    return utils.BYTE_FALSE, err
                }
                return utils.BYTE_TRUE, nil
            }
            //NOTE, you need to store the tx in this
            err = MakeTransaction(native, txParam, chainID)
            if err != nil {
                return utils.BYTE_FALSE, err
            }
            return utils.BYTE_TRUE, nil
        }
        ```
    2. This function first verifies the source chain tx, if it passes, the core parameters are sealed in a `ToMerkleValue` struct and its hash is saved into a `crossHashes` array:
        ```golang
        func MakeTransaction(service *native.NativeService, params *scom.MakeTxParam, fromChainID uint64) error {
            txHash := service.GetTx().Hash()
            merkleValue := &scom.ToMerkleValue{
                TxHash:      txHash.ToArray(),
                FromChainID: fromChainID,
                MakeTxParam: params,
            }

            sink := common.NewZeroCopySink(nil)
            merkleValue.Serialization(sink)
            err := PutRequest(service, merkleValue.TxHash, params.ToChainID, sink.Bytes())
            if err != nil {
                return fmt.Errorf("MakeTransaction, putRequest error:%s", err)
            }
            service.PutMerkleVal(sink.Bytes())
            chainIDBytes := utils.GetUint64Bytes(params.ToChainID)
            key := hex.EncodeToString(utils.ConcatKey(utils.CrossChainManagerContractAddress, []byte(scom.REQUEST), chainIDBytes, merkleValue.TxHash))
            scom.NotifyMakeProof(service, fromChainID, params.ToChainID, hex.EncodeToString(params.TxHash), key)
            return nil
        }
        ...
        func (this *NativeService) PutMerkleVal(data []byte) {
            this.crossHashes = append(this.crossHashes, merkle.HashLeaf(data))
        }
        ```    
    3. In the end the `crossHashes` happened in current block are used to computer `CrossStatesRoot`, which is the root of a merkle tree:
        ```golang
        func (this *LedgerStoreImp) executeBlock(block *types.Block) (result store.ExecuteResult, err error) {
            overlay := this.stateStore.NewOverlayDB()

            cache := storage.NewCacheDB(overlay)
            for _, tx := range block.Transactions {
                cache.Reset()
                notify, crossHashes, e := this.handleTransaction(overlay, cache, block, tx)
                if e != nil {
                    err = e
                    return
                }
                result.Notify = append(result.Notify, notify)
                result.CrossHashes = append(result.CrossHashes, crossHashes...)
            }
            if len(result.CrossHashes) != 0 {
                result.CrossStatesRoot = merkle.TreeHasher{}.HashFullTreeWithLeafHash(result.CrossHashes)
            } else {
                result.CrossStatesRoot = common.UINT256_EMPTY
            }
            result.Hash = overlay.ChangeHash()
            result.WriteSet = overlay.GetWriteSet()
            result.MerkleRoot = this.stateStore.GetStateMerkleRootWithNewHash(result.Hash)
            return
        }
        ```
    4. The above `ExecuteResult` is stored in a `PendingBlock`:
        ```golang
            ...
            execResult, err := self.db.ExecuteBlock(block.Block)
            if err != nil {
                log.Errorf("chainstore AddBlock GetBlockExecResult: %s", err)
                return fmt.Errorf("chainstore AddBlock GetBlockExecResult: %s", err)
            }
            self.pendingBlocks[blkNum] = &PendingBlock{block: block, execResult: &execResult, hasSubmitted: false}
            ...
        ```    
    5. When the next block is generated, `getCrossStateRoot` is called to fetch the above `CrossStatesRoot`, and put into the header:
        ```golang
        func (self *Server) constructBlock(blkNum uint32, prevBlkHash common.Uint256, txs []*types.Transaction,
            consensusPayload []byte, blocktimestamp uint32, nextBookkeeper common.Address) (*types.Block, error) {
            txHash := []common.Uint256{}
            for _, t := range txs {
                txHash = append(txHash, t.Hash())
            }
            lastBlock, _ := self.blockPool.getSealedBlock(blkNum - 1)
            if lastBlock == nil {
                log.Errorf("constructBlock getlastblock failed blknum:%d", blkNum-1)
                return nil, fmt.Errorf("constructBlock getlastblock failed blknum:%d", blkNum-1)
            }
            txRoot := common.ComputeMerkleRoot(txHash)

            blockRoot := ledger.DefLedger.GetBlockRootWithPreBlockHashes(blkNum-1, []common.Uint256{lastBlock.Block.Header.PrevBlockHash, prevBlkHash})
            crossStateRoot, err := self.blockPool.getCrossStatesRoot(blkNum - 1)
            if err != nil {
                return nil, fmt.Errorf("failed to GetCrossStatesRoot: %s,blkNum:%d", err, (blkNum - 1))
            }

            blkHeader := &types.Header{
                Version:          types.CURR_HEADER_VERSION,
                ChainID:          config.GetChainIdByNetId(config.DefConfig.P2PNode.NetworkId),
                PrevBlockHash:    prevBlkHash,
                TransactionsRoot: txRoot,
                CrossStateRoot:   crossStateRoot,
                BlockRoot:        blockRoot,
                Timestamp:        blocktimestamp,
                Height:           blkNum,
                NextBookkeeper:   nextBookkeeper,
                ConsensusData:    common.GetNonce(),
                ConsensusPayload: consensusPayload,
            }

            blk := &types.Block{
                Header:       blkHeader,
                Transactions: txs,
            }
            blkHash := blk.Hash()
            sig, err := signature.Sign(self.account, blkHash[:])
            if err != nil {
                return nil, fmt.Errorf("sign block failed, block hash:%s, error: %s", blkHash.ToHexString(), err)
            }
            blkHeader.Bookkeepers = []keypair.PublicKey{self.account.PublicKey}
            blkHeader.SigData = [][]byte{sig}

            return blk, nil
        }
        ```    
3. After the verifier set quorum confirms with a multi-signature, it is relayed to the target chain. The smart contract on the target chain verifies the fact and then transfers the target asset to the target address.
    1. There's a [`verifyHeaderAndExecuteTx`](https://github.com/polynetwork/eth-contracts/blob/master/contracts/core/cross_chain_manager/logic/EthCrossChainManager.sol#L191) function on the target chain:
    ```golang
    /* @notice              Verify Poly chain header and proof, execute the cross chain tx from Poly chain to Ethereum
    *  @param proof         Poly chain tx merkle proof
    *  @param rawHeader     The header containing crossStateRoot to verify the above tx merkle proof
    *  @param headerProof   The header merkle proof used to verify rawHeader
    *  @param curRawHeader  Any header in current epoch consensus of Poly chain
    *  @param headerSig     The coverted signature veriable for solidity derived from Poly chain consensus nodes' signature
    *                       used to verify the validity of curRawHeader
    *  @return              true or false
    */
    function verifyHeaderAndExecuteTx(bytes memory proof, bytes memory rawHeader, bytes memory headerProof, bytes memory curRawHeader,bytes memory headerSig) whenNotPaused public returns (bool){
        ECCUtils.Header memory header = ECCUtils.deserializeHeader(rawHeader);
        // Load ehereum cross chain data contract
        IEthCrossChainData eccd = IEthCrossChainData(EthCrossChainDataAddress);
        
        // Get stored consensus public key bytes of current poly chain epoch and deserialize Poly chain consensus public key bytes to address[]
        address[] memory polyChainBKs = ECCUtils.deserializeKeepers(eccd.getCurEpochConPubKeyBytes());

        uint256 curEpochStartHeight = eccd.getCurEpochStartHeight();

        uint n = polyChainBKs.length;
        if (header.height >= curEpochStartHeight) {
            // It's enough to verify rawHeader signature
            require(ECCUtils.verifySig(rawHeader, headerSig, polyChainBKs, n - ( n - 1) / 3), "Verify poly chain header signature failed!");
        } else {
            // We need to verify the signature of curHeader 
            require(ECCUtils.verifySig(curRawHeader, headerSig, polyChainBKs, n - ( n - 1) / 3), "Verify poly chain current epoch header signature failed!");

            // Then use curHeader.StateRoot and headerProof to verify rawHeader.CrossStateRoot
            ECCUtils.Header memory curHeader = ECCUtils.deserializeHeader(curRawHeader);
            bytes memory proveValue = ECCUtils.merkleProve(headerProof, curHeader.blockRoot);
            require(ECCUtils.getHeaderHash(rawHeader) == Utils.bytesToBytes32(proveValue), "verify header proof failed!");
        }
        
        // Through rawHeader.CrossStatesRoot, the toMerkleValue or cross chain msg can be verified and parsed from proof
        bytes memory toMerkleValueBs = ECCUtils.merkleProve(proof, header.crossStatesRoot);
        
        // Parse the toMerkleValue struct and make sure the tx has not been processed, then mark this tx as processed
        ECCUtils.ToMerkleValue memory toMerkleValue = ECCUtils.deserializeMerkleValue(toMerkleValueBs);
        require(!eccd.checkIfFromChainTxExist(toMerkleValue.fromChainID, Utils.bytesToBytes32(toMerkleValue.txHash)), "the transaction has been executed!");
        require(eccd.markFromChainTxExist(toMerkleValue.fromChainID, Utils.bytesToBytes32(toMerkleValue.txHash)), "Save crosschain tx exist failed!");
        
        // Ethereum ChainId is 2, we need to check the transaction is for Ethereum network
        require(toMerkleValue.makeTxParam.toChainId == chainId, "This Tx is not aiming at this network!");
        
        // Obtain the targeting contract, so that Ethereum cross chain manager contract can trigger the executation of cross chain tx on Ethereum side
        address toContract = Utils.bytesToAddress(toMerkleValue.makeTxParam.toContract);
        
        // only invoke PreWhiteListed Contract and method For Now
        require(whiteListContractMethodMap[toContract][toMerkleValue.makeTxParam.method],"Invalid to contract or method");

        //TODO: check this part to make sure we commit the next line when doing local net UT test
        require(_executeCrossChainTx(toContract, toMerkleValue.makeTxParam.method, toMerkleValue.makeTxParam.args, toMerkleValue.makeTxParam.fromContract, toMerkleValue.fromChainID), "Execute CrossChain Tx failed!");

        // Fire the cross chain event denoting the executation of cross chain tx is successful,
        // and this tx is coming from other public chains to current Ethereum network
        emit VerifyHeaderAndExecuteTxEvent(toMerkleValue.fromChainID, toMerkleValue.makeTxParam.toContract, toMerkleValue.txHash, toMerkleValue.makeTxParam.txHash);

        return true;
    }
    ```
    2. This function first checks the validity of the block header of the relay blockchain by checking whether it's signed by verifier set quorum, if it's valid, the `crossStatesRoot` field is trusted, through merkle proof, the `ToMerkleValue` is also valid, after that, by calling [`_executeCrossChainTx`](https://github.com/polynetwork/eth-contracts/blob/master/contracts/core/cross_chain_manager/logic/EthCrossChainManager.sol#L250) the target asset is transfered to the target address:
        ```solidity
        /* @notice                  Dynamically invoke the targeting contract, and trigger executation of cross chain tx on Ethereum side
        *  @param _toContract       The targeting contract that will be invoked by the Ethereum Cross Chain Manager contract
        *  @param _method           At which method will be invoked within the targeting contract
        *  @param _args             The parameter that will be passed into the targeting contract
        *  @param _fromContractAddr From chain smart contract address
        *  @param _fromChainId      Indicate from which chain current cross chain tx comes 
        *  @return                  true or false
        */
        function _executeCrossChainTx(address _toContract, bytes memory _method, bytes memory _args, bytes memory _fromContractAddr, uint64 _fromChainId) internal returns (bool){
            // Ensure the targeting contract gonna be invoked is indeed a contract rather than a normal account address
            require(Utils.isContract(_toContract), "The passed in address is not a contract!");
            bytes memory returnData;
            bool success;
            
            // The returnData will be bytes32, the last byte must be 01;
            (success, returnData) = _toContract.call(abi.encodePacked(bytes4(keccak256(abi.encodePacked(_method, "(bytes,bytes,uint64)"))), abi.encode(_args, _fromContractAddr, _fromChainId)));
            
            // Ensure the executation is successful
            require(success == true, "EthCrossChain call business contract failed");
            
            // Ensure the returned value is true
            require(returnData.length != 0, "No return value from business contract!");
            (bool res,) = ZeroCopySource.NextBool(returnData, 31);
            require(res == true, "EthCrossChain call business contract return is not true");
            
            return true;
        }
        ```    


# The hack

On August 10, 2021, a thus far unidentified attacker stole $612 million worth of cryptocurrency from cross-chain DeFi protocol Poly Network. Details [here](https://blog.chainalysis.com/reports/poly-network-hack-august-2021/).

The related contract change is [this PR](https://github.com/polynetwork/eth-contracts/pull/12).Most critically these two lines:

```solidity
function verifyHeaderAndExecuteTx(bytes memory proof, bytes memory rawHeader, bytes memory headerProof, bytes memory curRawHeader,bytes memory headerSig) whenNotPaused public returns (bool){
    ...
    // only invoke PreWhiteListed Contract and method For Now
    require(whiteListToContract[toContract],"Invalid to contract");
    require(whiteListMethod[toMerkleValue.makeTxParam.method],"Invalid method");
    ...
}
```

Before this PR, the `toContract` and `toMerkleValue.makeTxParam.method` are entirely under the control of the hacker. The hacker managed to change the verifier set via calling [`EthCrossChainData.putCurEpochConPubKeyBytes`](https://github.com/polynetwork/eth-contracts/blob/1ed0a30c9178e21bbe72ad2257c9f7c562bf7b6b/contracts/core/cross_chain_manager/data/EthCrossChainData.sol#L45):

```solidity
// Store Consensus book Keepers Public Key Bytes
function putCurEpochConPubKeyBytes(bytes memory curEpochPkBytes) public whenNotPaused onlyOwner returns (bool) {
    ConKeepersPkBytes = curEpochPkBytes;
    return true;
}
```

`toContract` can be set directly, and the method to call is calculated by:

```solidity
abi.encodePacked(bytes4(keccak256(abi.encodePacked(_method, "(bytes,bytes,uint64)")))
```

It's only 4 bytes, and hash collision is not hard in this case. More details [here](https://slowmist.medium.com/the-analysis-and-q-a-of-poly-network-being-hacked-8112a35beb39).