```k
module LIQUIDITY-BAKING-SPEC
  imports LIQUIDITY-BAKING-VERIFICATION
```

```k
  claim <k> now 0 => . ... </k>
        <mynow> #Timestamp(0) </mynow>

  claim <k> (now 0 => .) ~> #liquidityBakingCode ... </k>
        <mynow> #Timestamp(0) </mynow>

  claim <k> PUSH nat X:Int ; PUSH nat Y:Int ; PUSH nat Z:Int ; MUL ; EDIV ; IF_NONE {} { DUP ; CDR ; SWAP ; CAR ; PUSH nat 0 ; DIG 2 ; COMPARE ; EQ } => . ... </k>
        <stack> .Stack => [ bool ?_ ] ; [ nat (Z *Int Y) /Int X ] ; .Stack </stack>
    requires X >Int 0
     andBool Y >=Int 0
     andBool Z >=Int 0
```

```k
endmodule
```

## Add Liquidity

Allows callers to "mint" liquidity in excahnge for tez.
Caller gains liquidity equal to `tez * storage.lqtTotal / storage.xtzPool`.

-   Storage updates:

```
Storage( lqtTotal:  LqtTotal  => LqtTotal  + lqt_minted ;
         tokenPool: TokenPool => TokenPool + tokens_deposited ;
         xtzPool:   XtzPool   => XtzPool   + Tezos.amount
       )
```

-   Operations:

1. call to `transfer` entrypoint of liquidity address: Send tokens from sender to self.
2. call to `mintOrBurn` entrypoint of token address: Adds liquidity for the sender.

### Positive case

-   Preconditions

1.  the deadline has not passed (i.e. the `Tezos.now < input.deadline`)
2.  the tokens transferred is less than `input.maxTokensDeposited`
3.  the liquidity minted is more than `input.minLqtMinted`
4.  xtzPool is positive

```k
module LIQUIDITY-BAKING-ADDLIQUIDITY-POSITIVE-SPEC
  imports LIQUIDITY-BAKING-VERIFICATION
```

For performance reasons, we split the claim depending on the result of the `#ceildiv` operation.
We have one case for when `#ceildiv` results in an upwards rounding, and one for the case when the numerator is divisible by the denominator.

```k
  claim <k> #runProof(AddLiquidity(Owner, MinLqtMinted, MaxTokensDeposited, #Timestamp(Deadline))) => . </k>
        <stack> .Stack </stack>
        <mynow> #Timestamp(CurrentTime) </mynow>
        <myamount> #Mutez(Amount) </myamount>
        <myaddr> SelfAddress </myaddr>
        <lqtTotal> OldLqt => OldLqt +Int (Amount *Int OldLqt) /Int XtzAmount </lqtTotal>
        <xtzPool> #Mutez(XtzAmount => XtzAmount +Int Amount) </xtzPool>
        <tokenPool> TokenAmount => TokenAmount +Int #ceildiv(Amount *Int TokenAmount, XtzAmount) </tokenPool>
        <tokenAddress> TokenAddress:Address </tokenAddress>
        <lqtAddress> LqtAddress:Address </lqtAddress>
        <senderaddr> Sender </senderaddr>
        <nonce> #Nonce(Nonce => Nonce +Int 2) </nonce>
        <knownaddrs> KnownAddresses </knownaddrs>
        <paramtype> LocalEntrypoints </paramtype>
        <operations> _
                  => [ Transfer_tokens #TokenTransferData(Sender, SelfAddress, #ceildiv(Amount *Int TokenAmount, XtzAmount)) #Mutez(0) TokenAddress . %transfer    Nonce ] ;;
                     [ Transfer_tokens Pair ((Amount *Int OldLqt) /Int XtzAmount) Owner                                      #Mutez(0) LqtAddress   . %mintOrBurn (Nonce +Int 1) ] ;;
                     .InternalList
        </operations>
    requires CurrentTime <Int Deadline
     andBool XtzAmount   >Int 0
     andBool #ceildiv(Amount *Int TokenAmount, XtzAmount) <=Int MaxTokensDeposited
     andBool MinLqtMinted <=Int (Amount *Int OldLqt) /Int XtzAmount
     andBool #IsLegalMutezValue(Amount +Int XtzAmount)

     andBool #EntrypointExists(KnownAddresses, TokenAddress,   %transfer, #TokenTransferType())
     andBool #EntrypointExists(KnownAddresses,   LqtAddress, %mintOrBurn, pair int %quantity .AnnotationList address %target .AnnotationList)
     andBool #LocalEntrypointExists(LocalEntrypoints, %default, unit)
```

```k
endmodule
```

### Negative case

The execution fails if any of the following are true:
1.  the deadline has passed (i.e. the `Tezos.now < input.deadline`)
2.  the tokens transferred is more than `input.maxTokensDeposited`
3.  the liquidity minted is less than `input.minLqtMinted`
4.  xtzPool is 0

```k
module LIQUIDITY-BAKING-ADDLIQUIDITY-NEGATIVE-SPEC
  imports LIQUIDITY-BAKING-VERIFICATION
```

```k
  claim <k> #runProof(AddLiquidity(_Owner, MinLqtMinted, MaxTokensDeposited, #Timestamp(Deadline))) => Aborted(?_, ?_, ?_, ?_) </k>
        <stack> .Stack => ?_:FailedStack </stack>
        <mynow> #Timestamp(CurrentTime) </mynow>
        <myamount> #Mutez(Amount) </myamount>
        <lqtTotal> OldLqt </lqtTotal>
        <xtzPool> #Mutez(XtzAmount) </xtzPool>
        <tokenPool> TokenAmount </tokenPool>
    requires notBool( CurrentTime <Int Deadline
              andBool XtzAmount   >Int 0
              andBool #ceildiv(Amount *Int TokenAmount, XtzAmount) <=Int MaxTokensDeposited
              andBool MinLqtMinted <=Int (Amount *Int OldLqt) /Int XtzAmount
              andBool #IsLegalMutezValue(Amount +Int XtzAmount)
                    )
```

TODO: Deal with the case when the token contract or the liquidity token contract don't exist or have the wrong type.

```k
endmodule
```

## Remove Liquidity

The sender can burn liquidity tokens in exchange for tez and tokens sent to some address if the following conditions are satisfied:

1.  exactly 0 tez was transferred to this contract when it was invoked
2.  the current block time must be less than the deadline
3.  the amount of liquidity to be redeemed, when converted to xtz, is greater than `minXtzWithdrawn` and less than the amount of tez owned by the Liquidity Baking contract
4.  the amount of liquidity to be redeemed, when converted to tokens, is greater than `minTokensWithdrawn` and less than the amount of tokens owned by the Liquidity Baking contract
5.  the amount of liquidity to be redeemed is less than the total amount of liquidity and less than the amount of liquidity tokens owned by the sender
6.  the contract at address `storage.lqtAddress` has a well-formed `mintOrBurn` entrypoint
7.  the contract at address `storage.tokenAddress` has a well-formed `transfer` entrypoint

```k
module LIQUIDITY-BAKING-REMOVELIQUIDITY-POSITIVE-SPEC
  imports LIQUIDITY-BAKING-VERIFICATION

  claim <k> #runProof(RemoveLiquidity(To, LqtBurned, #Mutez(MinXtzWithdrawn), MinTokensWithdrawn, #Timestamp(Deadline))) => . </k>
        <stack> .Stack </stack>
        <mynow> #Timestamp(CurrentTime) </mynow>
        <myamount> #Mutez(0) </myamount>
        <myaddr> SelfAddress </myaddr>
        <lqtTotal> OldLqt => OldLqt -Int LqtBurned </lqtTotal>
        <xtzPool> #Mutez(XtzAmount => XtzAmount -Int (LqtBurned *Int XtzAmount) /Int OldLqt) </xtzPool>
        <tokenPool> TokenAmount => TokenAmount -Int (LqtBurned *Int TokenAmount) /Int OldLqt </tokenPool>
        <tokenAddress> TokenAddress:Address </tokenAddress>
        <lqtAddress> LqtAddress:Address </lqtAddress>
        <senderaddr> Sender </senderaddr>
        <nonce> #Nonce(Nonce => Nonce +Int 3) </nonce>
        <knownaddrs> KnownAddresses </knownaddrs>
        <paramtype> LocalEntrypoints </paramtype>
        <operations> _
                  => [ Transfer_tokens (Pair (0 -Int LqtBurned) Sender)                                              #Mutez(0)                                      LqtAddress . %mintOrBurn  Nonce         ] ;;
                     [ Transfer_tokens #TokenTransferData(SelfAddress, To, (LqtBurned *Int TokenAmount) /Int OldLqt) #Mutez(0)                                      TokenAddress . %transfer (Nonce +Int 1) ] ;;
                     [ Transfer_tokens Unit                                                                          #Mutez((LqtBurned *Int XtzAmount) /Int OldLqt) To . %default            (Nonce +Int 2) ] ;;
                     .InternalList
        </operations>
    requires CurrentTime <Int Deadline
     andBool OldLqt >Int 0
     andBool OldLqt >=Int LqtBurned
     andBool MinXtzWithdrawn <=Int (LqtBurned *Int XtzAmount) /Int OldLqt
     andBool MinTokensWithdrawn <=Int (LqtBurned *Int TokenAmount) /Int OldLqt
     andBool #IsLegalMutezValue((LqtBurned *Int XtzAmount) /Int OldLqt)
     andBool TokenAmount -Int (LqtBurned *Int TokenAmount) /Int OldLqt >=Int 0
     andBool #IsLegalMutezValue(XtzAmount   -Int (LqtBurned *Int   XtzAmount) /Int OldLqt)

     andBool #EntrypointExists(KnownAddresses, TokenAddress,   %transfer, #TokenTransferType())
     andBool #EntrypointExists(KnownAddresses,   LqtAddress, %mintOrBurn, pair int %quantity .AnnotationList address %target .AnnotationList)
     andBool #EntrypointExists(KnownAddresses,           To,    %default, #Type(unit))
     andBool #LocalEntrypointExists(LocalEntrypoints, %default, unit)
```

```k
endmodule
```

```k
module LIQUIDITY-BAKING-REMOVELIQUIDITY-NEGATIVE-SPEC
  imports LIQUIDITY-BAKING-VERIFICATION

  claim <k> #runProof(RemoveLiquidity(To, LqtBurned, #Mutez(MinXtzWithdrawn), MinTokensWithdrawn, #Timestamp(Deadline))) => Aborted(?_, ?_, ?_, ?_) </k>
        <stack> .Stack => ?_:FailedStack </stack>
        <mynow> #Timestamp(CurrentTime) </mynow>
        <myamount> #Mutez(0) </myamount>
        <myaddr> SelfAddress </myaddr>
        <lqtTotal> OldLqt </lqtTotal>
        <xtzPool> #Mutez(XtzAmount) </xtzPool>
        <tokenPool> TokenAmount </tokenPool>
        <tokenAddress> TokenAddress:Address </tokenAddress>
        <lqtAddress> LqtAddress:Address </lqtAddress>
        <senderaddr> Sender </senderaddr>
        <knownaddrs> KnownAddresses </knownaddrs>
        <paramtype> LocalEntrypoints </paramtype>
        <nonce> _ => ?_ </nonce>
    requires notBool( CurrentTime <Int Deadline
              andBool OldLqt >Int 0
              andBool OldLqt >=Int LqtBurned
              andBool MinXtzWithdrawn <=Int (LqtBurned *Int XtzAmount) /Int OldLqt
              andBool MinTokensWithdrawn <=Int (LqtBurned *Int TokenAmount) /Int OldLqt
              andBool #IsLegalMutezValue((LqtBurned *Int XtzAmount) /Int OldLqt)
              andBool TokenAmount -Int (LqtBurned *Int TokenAmount) /Int OldLqt >=Int 0
              andBool #IsLegalMutezValue(XtzAmount   -Int (LqtBurned *Int   XtzAmount) /Int OldLqt)
                    )

     andBool #EntrypointExists(KnownAddresses, TokenAddress,   %transfer, #TokenTransferType())
     andBool #EntrypointExists(KnownAddresses,   LqtAddress, %mintOrBurn, pair int %quantity .AnnotationList address %target .AnnotationList)
     andBool #EntrypointExists(KnownAddresses,           To,    %default, #Type(unit))
     andBool #LocalEntrypointExists(LocalEntrypoints, %default, unit)
endmodule
```

## Default

```k
module LIQUIDITY-BAKING-DEFAULT-SPEC
  imports LIQUIDITY-BAKING-VERIFICATION
```

Adds more money to the xtz reserves if the following conditions are satisifed:

1.  the updated token pool size is a legal mutez value

```k
  claim <k> #runProof(Default) => . </k>
        <stack> .Stack </stack>
        <myamount> #Mutez(Amount) </myamount>
        <xtzPool> #Mutez(XtzPool => XtzPool +Int Amount) </xtzPool>
     requires #IsLegalMutezValue(XtzPool +Int Amount)
```

If any of the conditions are not satisfied, the call fails.

```k
  claim <k> #runProof(Default) => Aborted(?_, ?_, ?_, ?_) </k>
        <stack> .Stack => ?_:FailedStack </stack>
        <myamount> #Mutez(Amount) </myamount>
        <xtzPool> #Mutez(XtzPool) </xtzPool>
    requires notBool #IsLegalMutezValue(XtzPool +Int Amount)
```

```k
endmodule
```

## Token To XTZ

```k
module LIQUIDITY-BAKING-TOKENTOXTZ-POSITIVE-SPEC
  imports LIQUIDITY-BAKING-VERIFICATION
```

A buyer sends tokens to the Liquidity Baking contract and receives a corresponding amount of xtz, if the following conditions are satisfied:

1.  the current block time must be less than the deadline
2.  exactly 0 tez was transferred to this contract when it was invoked
3.  the amount of tokens sold, when converted into xtz using the current exchange rate (and less the burn fee), is greater than `minXtzBought`
4.  the amount of tokens sold, when converted into xtz using the current exchange rate, it is less than or equal to the xtz owned by the Liquidity Baking contract

NOTE: because the burn fee calculation is performed in the mutez type, i.e., because the following calculation:

```
XtzBought * 999 / 1000
```

occurs in the mutez type, that means that the contract will revert unless the temporary value `XtzBought * 999` is a legal mutez value;
even though the final mutez value that is actually used is smaller than or equal to `XtzBought`, i.e., guaranteed to be a valid mutez value.

```k
  claim <k> #runProof(TokenToXtz(To, TokensSold:Int, #Mutez(MinXtzBought), #Timestamp(Deadline))) => . </k>
        <stack> .Stack </stack>
        <myamount> #Mutez(Amount) </myamount>
        <tokenAddress> TokenAddress:Address </tokenAddress>
        <xtzPool> #Mutez(XtzPool:Int => XtzPool -Int #CurrencyBought(XtzPool, TokenPool, TokensSold)) </xtzPool>
        <tokenPool> TokenPool:Int => TokenPool +Int TokensSold </tokenPool>
        <mynow> #Timestamp(CurrentTime) </mynow>
        <senderaddr> Sender </senderaddr>
        <paramtype> LocalEntrypoints </paramtype>
        <myaddr> SelfAddress:Address </myaddr>
        <nonce> #Nonce(N => N +Int 3) </nonce>
        <knownaddrs> KnownAddresses </knownaddrs>
        <operations> _
                  => [ Transfer_tokens #TokenTransferData(Sender, SelfAddress, TokensSold) #Mutez(0)                                                               TokenAddress . %transfer N        ]
                  ;; [ Transfer_tokens Unit                                                #Mutez(#XtzNetBurn(#CurrencyBought(XtzPool, TokenPool, TokensSold)))    To           . %default (N +Int 1)]
                  ;; [ Transfer_tokens Unit                                                #Mutez(#XtzBurnAmount(#CurrencyBought(XtzPool, TokenPool, TokensSold))) null_address . %default (N +Int 2)]
                  ;; .InternalList
        </operations>
     requires Amount ==Int 0
      andBool CurrentTime <Int Deadline
      andBool (TokenPool >Int 0 orBool TokensSold >Int 0)

      andBool #IsLegalMutezValue(MinXtzBought)
      andBool #IsLegalMutezValue(#CurrencyBought(XtzPool, TokenPool, TokensSold))
      andBool #IsLegalMutezValue(#CurrencyBought(XtzPool, TokenPool, TokensSold) *Int 999)
      andBool #IsLegalMutezValue(#XtzNetBurn(#CurrencyBought(XtzPool, TokenPool, TokensSold)))
      andBool #XtzNetBurn(#CurrencyBought(XtzPool, TokenPool, TokensSold)) >=Int MinXtzBought

      andBool #IsLegalMutezValue(#XtzBurnAmount(#CurrencyBought(XtzPool, TokenPool, TokensSold)))
      andBool #CurrencyBought(XtzPool, TokenPool, TokensSold) <=Int XtzPool
      andBool #IsLegalMutezValue(XtzPool -Int #CurrencyBought(XtzPool, TokenPool, TokensSold))

      andBool #EntrypointExists(KnownAddresses, TokenAddress, %transfer, #TokenTransferType())
      andBool #EntrypointExists(KnownAddresses, To,           %default,  #Type(unit))
      andBool #EntrypointExists(KnownAddresses, null_address, %default,  #Type(unit))
      andBool #LocalEntrypointExists(LocalEntrypoints, %default, unit)
endmodule
```

The following claims prove the negative case:

```k
module LIQUIDITY-BAKING-TOKENTOXTZ-NEGATIVE-1-SPEC
  imports LIQUIDITY-BAKING-VERIFICATION
  claim <k> #runProof(TokenToXtz(To, _TokensSold, #Mutez(_MinXtzBought), #Timestamp(Deadline))) => Aborted(?_, ?_, ?_, ?_) </k>
        <stack> .Stack => ?_:FailedStack </stack>
        <mynow> #Timestamp(CurrentTime) </mynow>
        <myamount> #Mutez(Amount) </myamount>
        <tokenAddress> TokenAddress:Address </tokenAddress>
        <paramtype> LocalEntrypoints </paramtype>
        <knownaddrs> KnownAddresses </knownaddrs>
     requires notBool ( Amount ==Int 0
                andBool CurrentTime <Int Deadline
                      )

      andBool #EntrypointExists(KnownAddresses, TokenAddress, %transfer, #TokenTransferType())
      andBool #EntrypointExists(KnownAddresses, To,           %default,  #Type(unit))
      andBool #EntrypointExists(KnownAddresses, null_address, %default,  #Type(unit))
      andBool #LocalEntrypointExists(LocalEntrypoints, %default, unit)
endmodule
```

```k
module LIQUIDITY-BAKING-TOKENTOXTZ-NEGATIVE-2-SPEC
  imports LIQUIDITY-BAKING-VERIFICATION
  claim <k> #runProof(TokenToXtz(To, TokensSold, #Mutez(_MinXtzBought), #Timestamp(Deadline))) => Aborted(?_, ?_, ?_, ?_) </k>
        <stack> .Stack => ?_:FailedStack </stack>
        <mynow> #Timestamp(CurrentTime) </mynow>
        <myamount> #Mutez(Amount) </myamount>
        <tokenAddress> TokenAddress:Address </tokenAddress>
        <xtzPool> #Mutez(_XtzPool) </xtzPool>
        <tokenPool> TokenPool </tokenPool>
        <paramtype> LocalEntrypoints </paramtype>
        <knownaddrs> KnownAddresses </knownaddrs>
     requires Amount ==Int 0
      andBool CurrentTime <Int Deadline
      andBool TokenPool ==Int 0 andBool TokensSold ==Int 0

      andBool #EntrypointExists(KnownAddresses, TokenAddress, %transfer, #TokenTransferType())
      andBool #EntrypointExists(KnownAddresses, To,           %default,  #Type(unit))
      andBool #EntrypointExists(KnownAddresses, null_address, %default,  #Type(unit))
      andBool #LocalEntrypointExists(LocalEntrypoints, %default, unit)
endmodule
```

```k
module LIQUIDITY-BAKING-TOKENTOXTZ-NEGATIVE-3-SPEC
  imports LIQUIDITY-BAKING-VERIFICATION
  claim <k> #runProof(TokenToXtz(To, TokensSold, #Mutez(MinXtzBought), #Timestamp(Deadline))) => Aborted(?_, ?_, ?_, ?_) </k>
        <stack> .Stack => ?_:FailedStack </stack>
        <mynow> #Timestamp(CurrentTime) </mynow>
        <myamount> #Mutez(Amount) </myamount>
        <tokenAddress> TokenAddress:Address </tokenAddress>
        <xtzPool> #Mutez(XtzPool) </xtzPool>
        <tokenPool> TokenPool </tokenPool>
        <paramtype> LocalEntrypoints </paramtype>
        <knownaddrs> KnownAddresses </knownaddrs>
        <nonce> #Nonce(_ => ?_) </nonce>
     requires Amount ==Int 0
      andBool CurrentTime <Int Deadline
      andBool ( TokenPool >Int 0 orBool TokensSold >Int 0 )
      andBool notBool( #IsLegalMutezValue(MinXtzBought)
               andBool #IsLegalMutezValue(#CurrencyBought(XtzPool, TokenPool, TokensSold))
               andBool #IsLegalMutezValue(#CurrencyBought(XtzPool, TokenPool, TokensSold) *Int 999)
               andBool #IsLegalMutezValue(#XtzNetBurn(#CurrencyBought(XtzPool, TokenPool, TokensSold)))
               andBool #XtzNetBurn(#CurrencyBought(XtzPool, TokenPool, TokensSold)) >=Int MinXtzBought
                     )

      andBool #EntrypointExists(KnownAddresses, TokenAddress, %transfer, #TokenTransferType())
      andBool #EntrypointExists(KnownAddresses, To,           %default,  #Type(unit))
      andBool #EntrypointExists(KnownAddresses, null_address, %default,  #Type(unit))
      andBool #LocalEntrypointExists(LocalEntrypoints, %default, unit)
endmodule
```

```k
module LIQUIDITY-BAKING-TOKENTOXTZ-NEGATIVE-4-SPEC
  imports LIQUIDITY-BAKING-VERIFICATION
  claim <k> #runProof(TokenToXtz(To, TokensSold, #Mutez(MinXtzBought), #Timestamp(Deadline))) => Aborted(?_, ?_, ?_, ?_) </k>
        <stack> .Stack => ?_:FailedStack </stack>
        <mynow> #Timestamp(CurrentTime) </mynow>
        <myamount> #Mutez(Amount) </myamount>
        <tokenAddress> TokenAddress:Address </tokenAddress>
        <xtzPool> #Mutez(XtzPool) </xtzPool>
        <tokenPool> TokenPool </tokenPool>
        <paramtype> LocalEntrypoints </paramtype>
        <knownaddrs> KnownAddresses </knownaddrs>
        <nonce> #Nonce(_ => ?_) </nonce>
     requires Amount ==Int 0
      andBool CurrentTime <Int Deadline
      andBool ( TokenPool >Int 0 orBool TokensSold >Int 0 )
      andBool #IsLegalMutezValue(MinXtzBought)
      andBool #IsLegalMutezValue(#CurrencyBought(XtzPool, TokenPool, TokensSold))
      andBool #IsLegalMutezValue(#CurrencyBought(XtzPool, TokenPool, TokensSold) *Int 999)
      andBool #IsLegalMutezValue(#XtzNetBurn(#CurrencyBought(XtzPool, TokenPool, TokensSold)))
      andBool #XtzNetBurn(#CurrencyBought(XtzPool, TokenPool, TokensSold)) >=Int MinXtzBought
      andBool notBool( #IsLegalMutezValue(#XtzBurnAmount(#CurrencyBought(XtzPool, TokenPool, TokensSold)))
               andBool #CurrencyBought(XtzPool, TokenPool, TokensSold) <=Int XtzPool
               andBool #IsLegalMutezValue(XtzPool -Int #CurrencyBought(XtzPool, TokenPool, TokensSold))
                     )

      andBool #EntrypointExists(KnownAddresses, TokenAddress, %transfer, #TokenTransferType())
      andBool #EntrypointExists(KnownAddresses, To,           %default,  #Type(unit))
      andBool #EntrypointExists(KnownAddresses, null_address, %default,  #Type(unit))
      andBool #LocalEntrypointExists(LocalEntrypoints, %default, unit)
endmodule
```

## XTZ To Token

A buyer sends xtz to the Liquidity Baking contract and receives a corresponding amount of tokens, if the following conditions are satisfied:

1.  the current block time must be less than the deadline
2.  when the `txn.amount` (in mutez) is converted into tokens using the current exchange rate, the purchased amount is greater than `minTokensBought`
3.  when the `txn.amount` (in mutez) is converted into tokens using the current exchange rate, it is less than or equal to the tokens owned by the Liquidity Baking contract

```k
module LIQUIDITY-BAKING-XTZTOTOKEN-POSITIVE-SPEC
  imports LIQUIDITY-BAKING-VERIFICATION
  claim <k> #runProof(XtzToToken(To, MinTokensBought, #Timestamp(Deadline))) => . </k>
        <stack> .Stack </stack>
        <paramtype> LocalEntrypoints </paramtype>
        <myamount> #Mutez(Amount) </myamount>
        <tokenAddress> TokenAddress </tokenAddress>
        <xtzPool> #Mutez(XtzPool => XtzPool +Int #XtzNetBurn(Amount)) </xtzPool>
        <tokenPool> TokenPool => TokenPool -Int #CurrencyBought(TokenPool, XtzPool, #XtzNetBurn(Amount)) </tokenPool>
        <mynow> #Timestamp(CurrentTime) </mynow>
        <myaddr> SelfAddress </myaddr>
        <nonce> #Nonce(N => N +Int 2) </nonce>
        <knownaddrs> KnownAddresses </knownaddrs>
        <operations> _
                  => [ Transfer_tokens #TokenTransferData(SelfAddress, To, #CurrencyBought(TokenPool, XtzPool, #XtzNetBurn(Amount))) #Mutez(0)                              TokenAddress . %transfer N        ]
                  ;; [ Transfer_tokens Unit                                                                                          #Mutez(absInt(#XtzBurnAmount(Amount))) null_address . %default (N +Int 1)]
                  ;; .InternalList
        </operations>
    requires CurrentTime <Int Deadline
     andBool XtzPool *Int 1000 +Int #mulDiv ( Amount , 999 , 1000 ) *Int 999 =/=Int 0
     andBool #CurrencyBought(TokenPool, XtzPool, #XtzNetBurn(Amount)) >=Int MinTokensBought
     andBool #CurrencyBought(TokenPool, XtzPool, #XtzNetBurn(Amount)) <=Int TokenPool
     andBool TokenPool -Int #CurrencyBought ( TokenPool , XtzPool , #XtzNetBurn(Amount) ) >=Int 0
     andBool #IsLegalMutezValue(XtzPool +Int #XtzNetBurn(Amount))
     andBool #IsLegalMutezValue(#XtzNetBurn(Amount))
     // andBool #IsLegalMutezValue(#XtzBurnAmount(Amount)) // this check can be inferred by the SMT solver so we don't explicitly need it

     andBool #EntrypointExists(KnownAddresses, TokenAddress, %transfer, #TokenTransferType())
     andBool #EntrypointExists(KnownAddresses, null_address, %default,  #Type(unit))
     andBool #LocalEntrypointExists(LocalEntrypoints, %default, unit)
endmodule
```

```k
module LIQUIDITY-BAKING-XTZTOTOKEN-NEGATIVE-SPEC
  imports LIQUIDITY-BAKING-VERIFICATION
  claim <k> #runProof(XtzToToken(_To, MinTokensBought, #Timestamp(Deadline))) => Aborted(?_, ?_, ?_, ?_) </k>
        <stack> .Stack => ?_:FailedStack </stack>
        <paramtype> LocalEntrypoints </paramtype>
        <myamount> #Mutez(Amount) </myamount>
        <tokenAddress> TokenAddress </tokenAddress>
        <xtzPool> #Mutez(XtzPool) </xtzPool>
        <tokenPool> TokenPool </tokenPool>
        <mynow> #Timestamp(CurrentTime) </mynow>
        <knownaddrs> KnownAddresses </knownaddrs>
        <nonce> #Nonce(_:Int => ?_:Int) </nonce>
        <operations> _ </operations>
    requires notBool ( CurrentTime <Int Deadline
               andBool XtzPool *Int 1000 +Int #mulDiv ( Amount , 999 , 1000 ) *Int 999 =/=Int 0
               andBool #CurrencyBought(TokenPool, XtzPool, #XtzNetBurn(Amount)) >=Int MinTokensBought
               andBool #CurrencyBought(TokenPool, XtzPool, #XtzNetBurn(Amount)) <=Int TokenPool
               andBool TokenPool -Int #CurrencyBought ( TokenPool , XtzPool , #XtzNetBurn(Amount) ) >=Int 0
               andBool #IsLegalMutezValue(XtzPool +Int #XtzNetBurn(Amount))
               andBool #IsLegalMutezValue(#XtzNetBurn(Amount))
               // andBool #IsLegalMutezValue(#XtzBurnAmount(Amount))
                     )

     andBool #EntrypointExists(KnownAddresses, TokenAddress, %transfer, #TokenTransferType())
     andBool #EntrypointExists(KnownAddresses, null_address, %default,  #Type(unit))
     andBool #LocalEntrypointExists(LocalEntrypoints, %default, unit)
endmodule
```

## Token To Token

```k
module LIQUIDITY-BAKING-TOKENTOTOKEN-POSITIVE-SPEC
  imports LIQUIDITY-BAKING-VERIFICATION
```

A buyer sends tokens to the Liquidity Baking contract, converts its to xtz, and then immediately purchases a corresponding amount of tokens from a Dexter contract (such that all transactions succeed or fail atomically), if the following conditions are satisfied:

1.  the current block time must be less than the deadline
2.  exactly 0 tez was transferred to this contract when it was invoked
3.  the contract at address `outputDexterContract` has a well-formed `xtz_to_token` entrypoint
4.  the amount of tokens sold, when converted into xtz using the current exchange rate, it is less than or equal to the xtz owned by the Liquidity Baking contract
5.  the contract at address `storage.tokenAddress` must have a well-formed `transfer` entry point

```k
  claim <k> #runProof(TokenToToken(OutputDexterContract, MinTokensBought, To, TokensSold, #Timestamp(Deadline:Int))) => . </k>
        <stack> .Stack </stack>
        <myamount> #Mutez(Amount) </myamount>
        <tokenAddress> TokenAddress </tokenAddress>
        <xtzPool> #Mutez(XtzPool => XtzPool -Int #CurrencyBought(XtzPool, TokenPool, TokensSold)) </xtzPool>
        <tokenPool> TokenPool => TokenPool +Int TokensSold </tokenPool>
        <mynow> #Timestamp(CurrentTime:Int) </mynow>
        <senderaddr> Sender </senderaddr>
        <myaddr> SelfAddress </myaddr>
        <paramtype> LocalEntrypoints </paramtype>
        <nonce> #Nonce(N => N +Int 3) </nonce>
        <knownaddrs> KnownAddresses </knownaddrs>
        <operations> _
                  => [ Transfer_tokens #TokenTransferData(Sender, SelfAddress, TokensSold) #Mutez(0)                                                               TokenAddress         . %transfer    N        ]
                  ;; [ Transfer_tokens Pair To Pair MinTokensBought #Timestamp(Deadline)   #Mutez(#XtzNetBurn(#CurrencyBought(XtzPool, TokenPool, TokensSold)))    OutputDexterContract . %xtzToToken (N +Int 1)]
                  ;; [ Transfer_tokens Unit                                                #Mutez(#XtzBurnAmount(#CurrencyBought(XtzPool, TokenPool, TokensSold))) null_address         . %default    (N +Int 2)]
                  ;; .InternalList
        </operations>
     requires Amount ==Int 0
      andBool CurrentTime <Int Deadline
      andBool (TokenPool =/=Int 0 orBool TokensSold =/=Int 0)

      andBool #IsLegalMutezValue(TokensSold *Int 999 *Int XtzPool)
      andBool #IsLegalMutezValue(#CurrencyBought(XtzPool, TokenPool, TokensSold) *Int 999)
      andBool #IsLegalMutezValue(#CurrencyBought(XtzPool, TokenPool, TokensSold))

      // NOTE: we do not need to check that #XtzNetBurn(#CurrencyBought(...)) == #CurrencyBought(...) * 999 / 1000 is valid _because_:
      //  1.  we know that #CurrencyBought(...) * 999 is valid
      //  2.  taking a valid mutez value and dividing it by a non-zero natural can never produce an overflow/underflow
      andBool #IsLegalMutezValue(#XtzBurnAmount(#CurrencyBought(XtzPool, TokenPool, TokensSold)))
      andBool #CurrencyBought(XtzPool, TokenPool, TokensSold) <=Int XtzPool
      andBool #IsLegalMutezValue(XtzPool -Int #CurrencyBought(XtzPool, TokenPool, TokensSold))

      andBool #EntrypointExists(KnownAddresses, TokenAddress,         %transfer,   #TokenTransferType())
      andBool #EntrypointExists(KnownAddresses, OutputDexterContract, %xtzToToken, #Type(pair address pair nat timestamp))
      andBool #EntrypointExists(KnownAddresses, null_address,         %default,    #Type(unit))
      andBool #LocalEntrypointExists(LocalEntrypoints, %default, unit)
```

```k
endmodule
```

```k
module LIQUIDITY-BAKING-TOKENTOTOKEN-NEGATIVE-SPEC
  imports LIQUIDITY-BAKING-VERIFICATION
```

```k
  claim <k> #runProof(TokenToToken(OutputDexterContract, _MinTokensBought, _To, TokensSold, #Timestamp(Deadline:Int))) => Aborted(?_, ?_, ?_, ?_) </k>
        <stack> .Stack => ?_:FailedStack </stack>
        <myamount> #Mutez(Amount) </myamount>
        <tokenAddress> TokenAddress </tokenAddress>
        <xtzPool> #Mutez(XtzPool) </xtzPool>
        <tokenPool> TokenPool </tokenPool>
        <mynow> #Timestamp(CurrentTime:Int) </mynow>
        <senderaddr> _Sender </senderaddr>
        <myaddr> _SelfAddress </myaddr>
        <paramtype> LocalEntrypoints </paramtype>
        <nonce> #Nonce(_:Int => ?_:Int) </nonce>
        <knownaddrs> KnownAddresses </knownaddrs>
        <operations> _ </operations>
     requires notBool ( Amount ==Int 0
                        andBool CurrentTime <Int Deadline
                        andBool (TokenPool =/=Int 0 orBool TokensSold =/=Int 0)

                        andBool #IsLegalMutezValue(TokensSold *Int 999 *Int XtzPool)
                        andBool #IsLegalMutezValue(#CurrencyBought(XtzPool, TokenPool, TokensSold) *Int 999)
                        andBool #IsLegalMutezValue(#CurrencyBought(XtzPool, TokenPool, TokensSold))

                        andBool #IsLegalMutezValue(#XtzBurnAmount(#CurrencyBought(XtzPool, TokenPool, TokensSold)))
                        andBool #CurrencyBought(XtzPool, TokenPool, TokensSold) <=Int XtzPool
                        andBool #IsLegalMutezValue(XtzPool -Int #CurrencyBought(XtzPool, TokenPool, TokensSold))
                      )
      andBool #EntrypointExists(KnownAddresses, TokenAddress,         %transfer,   #TokenTransferType())
      andBool #EntrypointExists(KnownAddresses, OutputDexterContract, %xtzToToken, #Type(pair address pair nat timestamp))
      andBool #EntrypointExists(KnownAddresses, null_address,         %default,    #Type(unit))
      andBool #LocalEntrypointExists(LocalEntrypoints, %default, unit)
```

```k
endmodule
```
