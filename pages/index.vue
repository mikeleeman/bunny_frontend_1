<template>
	<div class="p-5">
		User Accounts: {{accounts}}
		<br>
		BunnyGirlData: {{bunnyGirlData}}
		<br>
		Currency Selected: {{addressOfCurrencySelected}}
		<br>
		Level Selected: {{levelSelected}}
		<br>
		No. Of Tokens To Mint: {{noOfTokensToMint}}
		<br>
		User Balance: {{userBalanceText}}
		<br>
		Token Ids: {{tokenIds}}
		<br>
		Token URIs: {{tokenUris}}
		<br>
		Token Metadata: {{tokenMetadata}}
		<br>
		Mint Status: {{buyStatusText}}
		<br>
		<br>
		<b-form-group>
			<label>Level Selection:</label>
			<b-form-select v-model="levelSelected" :options="levelOptions"></b-form-select>
		</b-form-group>
		<b-form-group>
			<label>Currency Selection:</label>
			<b-form-select @change="populateUserBalanceText()" v-model="addressOfCurrencySelected" :options="currencyOptions"></b-form-select>
		</b-form-group>
		<b-form-group>
			<label>No Of Tokens To Mint:</label>
			<input class="form-control" v-model="noOfTokensToMint" type="number" :step="1" :min="1" :max="10"/>
		</b-form-group>
		<b-form-group>
			<b-button :disabled="isReady" @click="buy()">Buy</b-button>
		</b-form-group>
	</div>
</template>

<script>
import Web3 from 'web3';
import Web3Modal from "web3modal";
import { getProviderInfo } from "web3modal"
import WalletConnectProvider from "@walletconnect/web3-provider";
import axios from "axios";
export default {
	components:{},
	data(){
		return {
			web3: null,
			bunnyGirlContract: null,
			accounts: [],
			noOfTokensToMint: 1,
			addressOfCurrencySelected: null,
			levelSelected: null,
			isReady:false,
			userBalanceText: 0,
			tokenIds: [],
			tokenUris: [],
			tokenMetadata: [],
			buyStatusText: '',

			bunnyGirlData:{
				levelMultiplier: [],
				mintFee: '0',
			},
			levelOptions: [
				{ value: null, text: 'Please select an option' },
				{ value: 1, text: 'Level 1' },
				{ value: 2, text: 'Level 2' },
				{ value: 3, text: 'Level 3' },
			],
			currencyOptions: [
				{ value: null, text: 'Please select an option' },
				{ value: '0x0000000000000000000000000000000000000000', text: 'BNB' },
			],
		}
	},
	methods:{
		async populateUserBalanceText(){
			const web3 = this.web3;
			const BN = web3.utils.BN;
			if(this.addressOfCurrencySelected === '0x0000000000000000000000000000000000000000'){
				//get native token amount
				const userBalance = new BN(await web3.eth.getBalance(this.accounts[0]));
				//this.userBalanceText = web3.utils.fromWei(userBalance, 'ether');
				this.userBalanceText = userBalance.toString();
			}
		},
		

		async buy(){
			const web3 = this.web3;
			const BN = web3.utils.BN;
			const fee = this.calculateFee();
			console.log('fee: ',fee.toString());
			if(await this.isEnoughFunds(fee)){
				console.log('buy');
				await this.mintBunnyGirl();
			}else{
				alert('not enough fee');
			}
		},

		calculateFee(){
			const web3 = this.web3;
			const BN = web3.utils.BN;
			const multiplier = new BN(this.bunnyGirlData.levelMultiplier[this.levelSelected-1]);
			const baseFee = new BN(this.bunnyGirlData.mintFee);
			baseFee.imul(multiplier);
			baseFee.imuln(parseInt(this.noOfTokensToMint));
			baseFee.idivn(10000);
			console.log('calculated fee: ', baseFee.toString());
			return baseFee;
		},

		async mintBunnyGirl(){
			const bunnyGirlContract = this.bunnyGirlContract;
			console.log(this.noOfTokensToMint, this.addressOfCurrencySelected, this.levelSelected);
			const fee = this.calculateFee();

			bunnyGirlContract.methods.mintBunnyGirl(this.noOfTokensToMint, this.addressOfCurrencySelected, this.levelSelected)
			.send({from: this.accounts[0], gas: "7000000", value: fee})
			.on('transactionHash', (hash)=>{
				this.buyStatusText = 'Pending Receipt...';
				//display tx url for user to check
			}).once('receipt', async (receipt)=>{
				console.log('mintBunnyGirl receipt: ',receipt);
				alert('Successfully purchased token');
				this.buyStatusText = 'Success';
				
				this.extractTokenIds(receipt);
				await this.populateTokenUris();
				await this.populateTokenMetadata();
			}).on('error', (err, receipt)=>{
				this.buyStatusText = 'Error';
				console.error('mintBunnyGirl receipt error',);
				console.error(err, receipt);
				alert('something went wrong');
			});
		},

		extractTokenIds(receipt){
			let events = [];
			if(Array.isArray(receipt.events.Transfer)){
				events = receipt.events.Transfer;
			}else{
				events.push(receipt.events.Transfer);
			}

			console.log('events: ', events);

			events.forEach((item)=>{
				this.tokenIds.push(item.raw.topics[3]);
			});
		},

		async populateTokenUris(){
			const bunnyGirlContract = this.bunnyGirlContract;
			var i=0;
			while(i<this.tokenIds.length){
				const id = this.tokenIds[i];
				this.tokenUris.push(await bunnyGirlContract.methods.tokenURI(id).call());
				i++;
			}
		},

		async populateTokenMetadata(){
			var i=0;
			while(i<this.tokenUris.length){
				const uri = this.tokenUris[i];
				this.tokenMetadata.push(await axios.get(uri));
				i++;
			}
		},
		
		async initContractAndPopulateBunnyGirlData(){
			console.log('BunnyGirlNftAddress: ',this.$settings.BunnyGirlNftAddress);
			const web3 = this.web3;
			this.bunnyGirlContract = new web3.eth.Contract(this.$settings.BunnyGirlNftAbi, this.$settings.BunnyGirlNftAddress);
			const bunnyGirlContract = this.bunnyGirlContract;
			try{
				this.bunnyGirlData.levelMultiplier.push(await bunnyGirlContract.methods.levelMultiplier(1).call());
				this.bunnyGirlData.levelMultiplier.push(await bunnyGirlContract.methods.levelMultiplier(2).call());
				this.bunnyGirlData.levelMultiplier.push(await bunnyGirlContract.methods.levelMultiplier(3).call());
				console.log('levelMultiplier: ', this.bunnyGirlData.levelMultiplier);
				
				const mintFee = await bunnyGirlContract.methods.mintFee('0x0000000000000000000000000000000000000000').call();
				console.log('mintFee: ', mintFee);
				this.bunnyGirlData.mintFee = mintFee;
			}catch(err){
				console.error(err);
			}
		},

		async isEnoughFunds(fee){
			const web3 = this.web3;
			const BN = web3.utils.BN;
			let userBalance = new BN('0');
			if(this.addressOfCurrencySelected === '0x0000000000000000000000000000000000000000'){
				//get native token amount
				userBalance = new BN(await web3.eth.getBalance(this.accounts[0]));
				this.userBalanceText = userBalance.toString();
				//this.userBalanceText = web3.utils.fromWei(userBalance, 'ether');
			}
			console.log(userBalance.toString());
			//small amount of wei as margin
			const margin = new BN('100000000');
			userBalance.iadd(margin);
			if(fee.gt(userBalance)){
				alert('not enough funds, please add more funds');
				return false;
			}
			return true;
		},

		async initWeb3Modal(){
			const providerOptions = {
				walletconnect:{
					package: WalletConnectProvider,
					options: {
						rpc: {
							1: "https://mainnet.infura.io/v3/" + this.$settings.infuraId,
							4: 'https://rinkeby.infura.io/v3/' + this.$settings.infuraId,
							97: 'https://data-seed-prebsc-1-s1.binance.org:8545/',
							56: 'https://bsc-dataseed.binance.org/',	
						},                                
						chainId: 56,
					}
				},
			};

			console.log('infuraId is: ',this.$settings.infuraId);

			const web3Modal = new Web3Modal({
				cacheProvider: false,
				providerOptions,
			});

			const provider = await web3Modal.connect();
			console.log('provider object: ',provider);

			this.web3 = new Web3(provider);
			const web3 = this.web3;

			const providerInfo = getProviderInfo(provider);
			console.log('providerInfo: ', providerInfo);

			if(providerInfo.check==='isWalletConnect'){
				
				try{
					this.accounts = await web3.eth.getAccounts();
					console.log(web3.eth);
					const chainId = await web3.eth.getChainId();
					if(chainId !== this.$settings.chainId){
						const err =  new Error('wrong network');
						err.custom_status_code = 'wrong_network';
					}
				}catch(err){
					if(err.hasOwnProperty('custom_status_code')){
						if(err.custom_status_code === 'wrong_network'){
							alert('Wrong network, please change your network');
						}
					}
					console.error('unable to get accounts');
					console.error(err);
				}
			}
			else if(providerInfo.check==='isMetaMask'){
				//get metamask connected network ID

				const ethereum = provider;

				this.accounts = await ethereum.request({ method: 'eth_requestAccounts' });

				console.log('eth_requestAccounts response: ',this.accounts);

				const chainId = ethereum.request({method: 'eth_chainId'});

				if(chainId !== this.$settings.chainId){
					 try{
					 	await ethereum.request({method: 'wallet_switchEthereumChain', 
						 params: [
							 //convert chainId to hex format
							 {chainId: '0x' + this.$settings.chainId.toString(16)}
						 ]});
					 }catch(err){
						 console.warn(err);

						 if(err.hasOwnProperty('code')){
							 if(err.code === 4001){
								 console.error('User rejected the network change');
								 alert('User rejected the network change');
							 }
						 }

						 await ethereum.request({method: 'wallet_addEthereumChain', 
						 params: [
							 {chainId: '0x' + this.$settings.chainId.toString(16), rpcUrls: 
								[ this.$settings.isUsingInfuraRpcUrl 
								? this.$settings.infuraRpcUrlPath + this.$settings.infuraId
								: this.$settings.rpcUrl]
							 }
						 ]});
						 
					 }
				}
			}

			provider.on("accountsChanged", (accounts) => {
				console.log(accounts);
				this.initWeb3Modal();
			});

			provider.on("chainChanged", (chainId) => {
				console.log('chainChange to: ',chainId);
				console.log('this.$settings.chainId:', '0x'+this.$settings.chainId.toString(16));
				if(chainId !== '0x'+this.$settings.chainId.toString(16)){
					alert('Wrong network, please change your network');
					//TODO: refactor into Swal, when close the dialog, init web3modal again
				}
				this.initWeb3Modal();
			});

			provider.on("connect", () => {
				console.log("connect");
			});

			provider.on("disconnect", (code, reason) => {
				console.log(code, reason);
			});

		},

		/*async approvePurchase(){
			const web3 = this.web3;
			const bunnyGirlContract = this.bunnyGirlContract;
			const fee = this.calculateFee();

			bunnyGirlContract.methods.approve(this.$settings.BunnyGirlNftAddress, fee).
			send({from: this.accounts[0]})
			.on('transactionHash', (hash)=>{
				//display tx url for user to check
			}).on('receipt', (receipt)=>{
				alert('Approve Success');
				console.log(receipt);
			}).on('error', (err, receipt)=>{
				if(err.hasOwnProperty('code')){
					if(err.code === 4001){
						alert('User rejected approval');
						return;
					}
				}
				alert('Approve Failed, Please Try Again');
				console.error(err);
				console.error(receipt);
			});
		},*/
	},
	//
	async mounted(){
		try{
			await this.initWeb3Modal();
			await this.initContractAndPopulateBunnyGirlData();
		}catch(err){
			console.error(err);
		}
	}
}
</script>
