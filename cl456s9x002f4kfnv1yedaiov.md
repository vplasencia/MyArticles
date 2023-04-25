---
title: "How to create a Zero Knowledge DApp: From zero to production"
datePublished: Wed Jun 08 2022 06:05:00 GMT+0000 (Coordinated Universal Time)
cuid: cl456s9x002f4kfnv1yedaiov
slug: how-to-create-a-zero-knowledge-dapp-from-zero-to-production
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1654662460748/il_O848lH.png
tags: web-application, web3, zero-knowledge

---

This is a step-by-step guide on how to build a [Zero Knowledge](https://en.wikipedia.org/wiki/Zero-knowledge_proof) (zk) [Decentralized Application](https://en.wikipedia.org/wiki/Decentralized_application) (DApp) from zero to production.

The goal is to explain the flow of a zk dapp and deploy it so that users can use it.

We will create a zk DApp to prove that someone knows how to solve a sudoku game, without revealing the answer.

We will use Circom (for circuits), Solidity (for smart contracts) and Javascript (for the frontend).

I will cover circuits and smart contracts testing.

I will give some tips that I use to build and organize this kind of projects.

To build the zk dapp we will use Groth16 and then do the same but with Plonk.

If you want to understand more about some topics, click on the links.

All the code is open source so you can read the article and look at the code:

* zkSudoku using Groth16: [https://github.com/vplasencia/zkSudoku](https://github.com/vplasencia/zkSudoku)
    
* zkSudoku using Plonk: [https://github.com/vplasencia/zkSudoku-plonk](https://github.com/vplasencia/zkSudoku-plonk)
    

We will deploy smart contracts on [Sepolia](https://sepolia.etherscan.io/) and the frontend on [Vercel](https://vercel.com/).

## Install dependencies

These are some important dependencies that we will use:

* [npm](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm/)
    
* [yarn](https://yarnpkg.com/getting-started/install)
    
* [circom](https://docs.circom.io/getting-started/installation/)
    
* [snarkjs](https://github.com/iden3/snarkjs#install-snarkjs)
    

## Circuits

### 1\. Create the circuit

* Create the `zkSudoku` folder:
    

```bash
mkdir zkSudoku
```

* Go inside the `zkSudoku` folder:
    

```bash
cd zkSudoku
```

* Create the `circuits` folder:
    

```bash
mkdir circuits
```

* Go inside the `circuits` folder:
    

```bash
cd circuits
```

* Create the `package.json` file:
    

```bash
yarn init -y
```

* Add the `circomlib` library to use some circuits from there:
    

```bash
yarn add circomlib
```

**Note:** To know more about the `circomlib` library read the [circomlib documentation](https://github.com/iden3/circomlib).

* Create the `sudoku` folder to write the circuit there:
    

```bash
mkdir sudoku
```

* Go inside the `sudoku` folder:
    

```bash
cd sudoku
```

* Open a code editor inside the `sudoku` folder.
    

**Note:** I use Visual Studio Code. To open Visual Studio Code, run:

```bash
code .
```

* Create the `sudoku.circom` file and add the circom code:
    

```bash
pragma circom 2.0.0;

include "../node_modules/circomlib/circuits/comparators.circom";

template Sudoku() {
    signal input unsolved[9][9];
    signal input solved[9][9];


    // Check if the numbers of the solved sudoku are >=1 and <=9
    // Each number in the solved sudoku is checked to see if it is >=1 and <=9

    component getone[9][9];
    component letnine[9][9];


    for (var i = 0; i < 9; i++) {
       for (var j = 0; j < 9; j++) {
           letnine[i][j] = LessEqThan(32);
           letnine[i][j].in[0] <== solved[i][j];
           letnine[i][j].in[1] <== 9;

           getone[i][j] = GreaterEqThan(32);
           getone[i][j].in[0] <== solved[i][j];
           getone[i][j].in[1] <== 1;

           letnine[i][j].out ===  getone[i][j].out;
        }
    }


    // Check if unsolved is the initial state of solved
    // If unsolved[i][j] is not zero, it means that solved[i][j] is equal to unsolved[i][j]
    // If unsolved[i][j] is zero, it means that solved [i][j] is different from unsolved[i][j]

    component ieBoard[9][9];
    component izBoard[9][9];

    for (var i = 0; i < 9; i++) {
       for (var j = 0; j < 9; j++) {
            ieBoard[i][j] = IsEqual();
            ieBoard[i][j].in[0] <== solved[i][j];
            ieBoard[i][j].in[1] <== unsolved[i][j];

            izBoard[i][j] = IsZero();
            izBoard[i][j].in <== unsolved[i][j];

            ieBoard[i][j].out === 1 - izBoard[i][j].out;
        }
    }


    // Check if each row in solved has all the numbers from 1 to 9, both included
    // For each element in solved, check that this element is not equal
    // to previous elements in the same row

    component ieRow[324];

    var indexRow = 0;


    for (var i = 0; i < 9; i++) {
       for (var j = 0; j < 9; j++) {
            for (var k = 0; k < j; k++) {
                ieRow[indexRow] = IsEqual();
                ieRow[indexRow].in[0] <== solved[i][k];
                ieRow[indexRow].in[1] <== solved[i][j];
                ieRow[indexRow].out === 0;
                indexRow ++;
            }
        }
    }


    // Check if each column in solved has all the numbers from 1 to 9, both included
    // For each element in solved, check that this element is not equal
    // to previous elements in the same column

    component ieCol[324];

    var indexCol = 0;


    for (var i = 0; i < 9; i++) {
       for (var j = 0; j < 9; j++) {
            for (var k = 0; k < i; k++) {
                ieCol[indexCol] = IsEqual();
                ieCol[indexCol].in[0] <== solved[k][j];
                ieCol[indexCol].in[1] <== solved[i][j];
                ieCol[indexCol].out === 0;
                indexCol ++;
            }
        }
    }


    // Check if each square in solved has all the numbers from 1 to 9, both included
    // For each square and for each element in each square, check that the
    // element is not equal to previous elements in the same square

    component ieSquare[324];

    var indexSquare = 0;

    for (var i = 0; i < 9; i+=3) {
       for (var j = 0; j < 9; j+=3) {
            for (var k = i; k < i+3; k++) {
                for (var l = j; l < j+3; l++) {
                    for (var m = i; m <= k; m++) {
                        for (var n = j; n < l; n++){
                            ieSquare[indexSquare] = IsEqual();
                            ieSquare[indexSquare].in[0] <== solved[m][n];
                            ieSquare[indexSquare].in[1] <== solved[k][l];
                            ieSquare[indexSquare].out === 0;
                            indexSquare ++;
                        }
                    }
                }
            }
        }
    }

}

// unsolved is a public input signal. It is the unsolved sudoku
component main {public [unsolved]} = Sudoku();
```

### 2\. Compile the circuit

* Create a `compile.sh` file to use it every time you want to compile the circuit.
    

**Note:** All the `.sh` files created inside the `zkSudoku/circuits/sudoku` folder, are generic, so you can use them in your circuits.

You can use the `compile.sh` script by running the file and passing it the name of the circuit: `./compile.sh sudoku`. Or you can edit the `CIRCUIT` variable inside the `compile.sh` file with the name of your circuit and run: `./compile.sh`. The first time you run the script, you should run: `chmod u+x compile.sh`.

```bash
#!/bin/bash

# Variable to store the name of the circuit
CIRCUIT=sudoku

# In case there is a circuit name as input
if [ "$1" ]; then
    CIRCUIT=$1
fi

# Compile the circuit
circom ${CIRCUIT}.circom --r1cs --wasm --sym --c
```

**Note:** To learn how the above file was created, read the [snarkjs documentation](https://github.com/iden3/snarkjs).

* Run the file.
    

Run the first time:

```bash
chmod u+x compile.sh
```

And after that, you can always run:

```bash
./compile.sh
```

You should see something like this:

![CompileCircuitImage.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654638604295/v1WTLgZYo.png align="left")

### 3\. Add the input file

* Create the `input.json` file and add to it:
    

```json
{
  "unsolved": [
    [0, 0, 0, 0, 0, 6, 0, 0, 0],
    [0, 0, 7, 2, 0, 0, 8, 0, 0],
    [9, 0, 6, 8, 0, 0, 0, 1, 0],
    [3, 0, 0, 7, 0, 0, 0, 2, 9],
    [0, 0, 0, 0, 0, 0, 0, 0, 0],
    [4, 0, 0, 5, 0, 0, 0, 7, 0],
    [6, 5, 0, 1, 0, 0, 0, 0, 0],
    [8, 0, 1, 0, 5, 0, 3, 0, 0],
    [7, 9, 2, 0, 0, 0, 0, 0, 4]
  ],
  "solved": [
    [1, 8, 4, 3, 7, 6, 2, 9, 5],
    [5, 3, 7, 2, 9, 1, 8, 4, 6],
    [9, 2, 6, 8, 4, 5, 7, 1, 3],
    [3, 6, 5, 7, 1, 8, 4, 2, 9],
    [2, 7, 8, 4, 6, 9, 5, 3, 1],
    [4, 1, 9, 5, 3, 2, 6, 7, 8],
    [6, 5, 3, 1, 2, 4, 9, 8, 7],
    [8, 4, 1, 9, 5, 7, 3, 6, 2],
    [7, 9, 2, 6, 8, 3, 1, 5, 4]
  ]
}
```

### 4\. Generate the witness

* Create the `generateWitness.sh` file and add to it:
    

```bash
#!/bin/bash

# Variable to store the name of the circuit
CIRCUIT=sudoku

# In case there is a circuit name as input
if [ "$1" ]; then
    CIRCUIT=$1
fi

# Compile the circuit
circom ${CIRCUIT}.circom --r1cs --wasm --sym --c

# Generate the witness.wtns
node ${CIRCUIT}_js/generate_witness.js ${CIRCUIT}_js/${CIRCUIT}.wasm input.json ${CIRCUIT}_js/witness.wtns
```

**Note:** To learn how the above file was created, read the [snarkjs documentation](https://github.com/iden3/snarkjs).

* Run the file
    

Run the first time:

```bash
chmod u+x generateWitness.sh
```

And after that, you can always run:

```bash
./generateWitness.sh
```

You can use the `generateWitness.sh` script by running the file and passing it the name of the circuit: `./generateWitness.sh sudoku`. Or you can edit the `CIRCUIT` variable inside the `generateWitness.sh` file with the name of your circuit and run: `./generateWitness.sh`. The first time you run the script, you should run: `chmod u+x generateWitness.sh`.

When you run the script you will see the `witness.wtns` file inside the `sudoku_js` folder.

### 5\. Generate all the necessary files

Create the `executeGroth16.sh` file and add to it:

This is a generic file, it can be used with any circuit (it uses groth16). For example, if you want to run a circuit called `circuit.circom` with the ptau 12, you can run the script like this: `./executeGroth16.sh circuit 12` or you can also modify the `CIRCUIT` and `PTAU` variables like this: `CIRCUIT=circuit` and `PTAU=12`.

```bash
#!/bin/bash

# Variable to store the name of the circuit
CIRCUIT=sudoku

# Variable to store the number of the ptau file
PTAU=14

# In case there is a circuit name as an input
if [ "$1" ]; then
    CIRCUIT=$1
fi

# In case there is a ptau file number as an input
if [ "$2" ]; then
    PTAU=$2
fi

# Check if the necessary ptau file already exists. If it does not exist, it will be downloaded from the data center
if [ -f ./ptau/powersOfTau28_hez_final_${PTAU}.ptau ]; then
    echo "----- powersOfTau28_hez_final_${PTAU}.ptau already exists -----"
else
    echo "----- Download powersOfTau28_hez_final_${PTAU}.ptau -----"
    wget -P ./ptau https://hermez.s3-eu-west-1.amazonaws.com/powersOfTau28_hez_final_${PTAU}.ptau
fi

# Compile the circuit
circom ${CIRCUIT}.circom --r1cs --wasm --sym --c

# Generate the witness.wtns
node ${CIRCUIT}_js/generate_witness.js ${CIRCUIT}_js/${CIRCUIT}.wasm input.json ${CIRCUIT}_js/witness.wtns

echo "----- Generate .zkey file -----"
# Generate a .zkey file that will contain the proving and verification keys together with all phase 2 contributions
snarkjs groth16 setup ${CIRCUIT}.r1cs ptau/powersOfTau28_hez_final_${PTAU}.ptau ${CIRCUIT}_0000.zkey

echo "----- Contribute to the phase 2 of the ceremony -----"
# Contribute to the phase 2 of the ceremony
snarkjs zkey contribute ${CIRCUIT}_0000.zkey ${CIRCUIT}_final.zkey --name="1st Contributor Name" -v -e="some random text"

echo "----- Export the verification key -----"
# Export the verification key
snarkjs zkey export verificationkey ${CIRCUIT}_final.zkey verification_key.json

echo "----- Generate zk-proof -----"
# Generate a zk-proof associated to the circuit and the witness. This generates proof.json and public.json
snarkjs groth16 prove ${CIRCUIT}_final.zkey ${CIRCUIT}_js/witness.wtns proof.json public.json

echo "----- Verify the proof -----"
# Verify the proof
snarkjs groth16 verify verification_key.json public.json proof.json

echo "----- Generate Solidity verifier -----"
# Generate a Solidity verifier that allows verifying proofs on Ethereum blockchain
snarkjs zkey export solidityverifier ${CIRCUIT}_final.zkey ${CIRCUIT}Verifier.sol
# Update the solidity version in the Solidity verifier
sed -i 's/0.6.11;/0.8.4;/g' ${CIRCUIT}Verifier.sol
# Update the contract name in the Solidity verifier
sed -i "s/contract Verifier/contract ${CIRCUIT^}Verifier/g" ${CIRCUIT}Verifier.sol

echo "----- Generate and print parameters of call -----"
# Generate and print parameters of call
snarkjs generatecall | tee parameters.txt
```

**Note:** To learn how the above file was created, read the [snarkjs documentation](https://github.com/iden3/snarkjs).

* Run the file
    

Run the first time:

```bash
chmod u+x executeGroth16.sh
```

And after that, you can always run:

```bash
./executeGroth16.sh
```

When you run the script if everything was correct, you will see `[INFO] snarkJS: OK!`:

![ExecuteFileImage.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654638636560/sv9gJ5mQQ.png align="left")

### 6\. Test circuits

* Go inside the `circuits` folder and open a code editor.
    

**Note:** I use Visual Studio Code. To open Visual Studio Code, run:

```bash
code .
```

* Install Dependencies:
    
    * Install [chai](https://www.chaijs.com/) as a dev dependency:
        
    
    ```bash
    yarn add -D chai
    ```
    
    * Install [circom\_tester](https://github.com/iden3/circom_tester):
        
    
    ```bash
    yarn add circom_tester
    ```
    
* Create a `test` folder.
    
* Inside the `test` folder, create a `circuits.js` file and add to it:
    

```javascript
const { assert } = require("chai");
const wasm_tester = require("circom_tester").wasm;

describe("Sudoku circuit", function () {
  let sudokuCircuit;

  before(async function () {
    sudokuCircuit = await wasm_tester("sudoku/sudoku.circom");
  });

  it("Should generate the witness successfully", async function () {
    let input = {
      unsolved: [
        [0, 0, 0, 0, 0, 6, 0, 0, 0],
        [0, 0, 7, 2, 0, 0, 8, 0, 0],
        [9, 0, 6, 8, 0, 0, 0, 1, 0],
        [3, 0, 0, 7, 0, 0, 0, 2, 9],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [4, 0, 0, 5, 0, 0, 0, 7, 0],
        [6, 5, 0, 1, 0, 0, 0, 0, 0],
        [8, 0, 1, 0, 5, 0, 3, 0, 0],
        [7, 9, 2, 0, 0, 0, 0, 0, 4],
      ],
      solved: [
        [1, 8, 4, 3, 7, 6, 2, 9, 5],
        [5, 3, 7, 2, 9, 1, 8, 4, 6],
        [9, 2, 6, 8, 4, 5, 7, 1, 3],
        [3, 6, 5, 7, 1, 8, 4, 2, 9],
        [2, 7, 8, 4, 6, 9, 5, 3, 1],
        [4, 1, 9, 5, 3, 2, 6, 7, 8],
        [6, 5, 3, 1, 2, 4, 9, 8, 7],
        [8, 4, 1, 9, 5, 7, 3, 6, 2],
        [7, 9, 2, 6, 8, 3, 1, 5, 4],
      ],
    };
    const witness = await sudokuCircuit.calculateWitness(input);
    await sudokuCircuit.assertOut(witness, {});
  });
  it("Should fail because there is a number out of bounds", async function () {
    // The number 10 in the first row of solved is > 9
    let input = {
      unsolved: [
        [0, 0, 0, 0, 0, 6, 0, 0, 0],
        [0, 0, 7, 2, 0, 0, 8, 0, 0],
        [9, 0, 6, 8, 0, 0, 0, 1, 0],
        [3, 0, 0, 7, 0, 0, 0, 2, 9],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [4, 0, 0, 5, 0, 0, 0, 7, 0],
        [6, 5, 0, 1, 0, 0, 0, 0, 0],
        [8, 0, 1, 0, 5, 0, 3, 0, 0],
        [7, 9, 2, 0, 0, 0, 0, 0, 4],
      ],
      solved: [
        [1, 8, 4, 3, 7, 6, 2, 9, 10],
        [5, 3, 7, 2, 9, 1, 8, 4, 6],
        [9, 2, 6, 8, 4, 5, 7, 1, 3],
        [3, 6, 5, 7, 1, 8, 4, 2, 9],
        [2, 7, 8, 4, 6, 9, 5, 3, 1],
        [4, 1, 9, 5, 3, 2, 6, 7, 8],
        [6, 5, 3, 1, 2, 4, 9, 8, 7],
        [8, 4, 1, 9, 5, 7, 3, 6, 2],
        [7, 9, 2, 6, 8, 3, 1, 5, 4],
      ],
    };
    try {
      await sudokuCircuit.calculateWitness(input);
    } catch (err) {
      // console.log(err);
      assert(err.message.includes("Assert Failed"));
    }
  });
  it("Should fail because unsolved is not the initial state of solved", async function () {
    // unsolved is not the initial state of solved
    let input = {
      unsolved: [
        [0, 0, 0, 0, 0, 6, 0, 0, 0],
        [0, 0, 7, 2, 0, 0, 8, 0, 0],
        [9, 0, 6, 8, 0, 0, 0, 1, 0],
        [3, 0, 0, 7, 0, 0, 0, 2, 9],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [4, 0, 0, 5, 0, 0, 0, 7, 0],
        [6, 5, 0, 1, 0, 0, 0, 0, 0],
        [8, 0, 1, 0, 5, 0, 3, 0, 0],
        [7, 9, 2, 0, 0, 0, 0, 0, 4],
      ],
      solved: [
        [1, 2, 7, 5, 8, 4, 6, 9, 3],
        [8, 5, 6, 3, 7, 9, 1, 2, 4],
        [3, 4, 9, 6, 2, 1, 8, 7, 5],
        [4, 7, 1, 9, 5, 8, 2, 3, 6],
        [2, 6, 8, 7, 1, 3, 5, 4, 9],
        [9, 3, 5, 4, 6, 2, 7, 1, 8],
        [5, 8, 3, 2, 9, 7, 4, 6, 1],
        [7, 1, 4, 8, 3, 6, 9, 5, 2],
        [6, 9, 2, 1, 4, 5, 3, 8, 7],
      ],
    };
    try {
      await sudokuCircuit.calculateWitness(input);
    } catch (err) {
      // console.log(err);
      assert(err.message.includes("Assert Failed"));
    }
  });
  it("Should fail due to repeated numbers in a row", async function () {
    // The number 1 in the first row of solved is twice
    let input = {
      unsolved: [
        [0, 0, 0, 0, 0, 6, 0, 0, 0],
        [0, 0, 7, 2, 0, 0, 8, 0, 0],
        [9, 0, 6, 8, 0, 0, 0, 1, 0],
        [3, 0, 0, 7, 0, 0, 0, 2, 9],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [4, 0, 0, 5, 0, 0, 0, 7, 0],
        [6, 5, 0, 1, 0, 0, 0, 0, 0],
        [8, 0, 1, 0, 5, 0, 3, 0, 0],
        [7, 9, 2, 0, 0, 0, 0, 0, 4],
      ],
      solved: [
        [1, 8, 4, 3, 7, 6, 2, 9, 1],
        [5, 3, 7, 2, 9, 1, 8, 4, 6],
        [9, 2, 6, 8, 4, 5, 7, 1, 3],
        [3, 6, 5, 7, 1, 8, 4, 2, 9],
        [2, 7, 8, 4, 6, 9, 5, 3, 1],
        [4, 1, 9, 5, 3, 2, 6, 7, 8],
        [6, 5, 3, 1, 2, 4, 9, 8, 7],
        [8, 4, 1, 9, 5, 7, 3, 6, 2],
        [7, 9, 2, 6, 8, 3, 1, 5, 4],
      ],
    };
    try {
      await sudokuCircuit.calculateWitness(input);
    } catch (err) {
      // console.log(err);
      assert(err.message.includes("Assert Failed"));
    }
  });
  it("Should fail due to repeated numbers in a column", async function () {
    // The number 4 in the first column of solved is twice and the number 7 in the last column of solved is twice too
    let input = {
      unsolved: [
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
      ],
      solved: [
        [1, 8, 4, 3, 7, 6, 2, 9, 5],
        [5, 3, 7, 2, 9, 1, 8, 4, 6],
        [9, 2, 6, 8, 4, 5, 7, 1, 3],
        [3, 6, 5, 7, 1, 8, 4, 2, 9],
        [2, 7, 8, 4, 6, 9, 5, 3, 1],
        [4, 1, 9, 5, 3, 2, 6, 7, 8],
        [6, 5, 3, 1, 2, 4, 9, 8, 7],
        [8, 4, 1, 9, 5, 7, 3, 6, 2],
        [4, 9, 2, 6, 8, 3, 1, 5, 7],
      ],
    };
    try {
      await sudokuCircuit.calculateWitness(input);
    } catch (err) {
      // console.log(err);
      assert(err.message.includes("Assert Failed"));
    }
  });
  it("Should fail due to repeated numbers in a square", async function () {
    // The number 1 in the first square (top-left) of solved is twice
    let input = {
      unsolved: [
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0, 0, 0, 0, 0],
      ],
      solved: [
        [1, 8, 4, 3, 7, 6, 2, 9, 5],
        [5, 3, 7, 2, 9, 1, 8, 4, 6],
        [9, 2, 1, 8, 4, 5, 7, 6, 3],
        [3, 6, 5, 7, 1, 8, 4, 2, 9],
        [2, 7, 8, 4, 6, 9, 5, 3, 1],
        [4, 1, 9, 5, 3, 2, 6, 7, 8],
        [6, 5, 3, 1, 2, 4, 9, 8, 7],
        [8, 4, 6, 9, 5, 7, 3, 1, 2],
        [7, 9, 2, 6, 8, 3, 1, 5, 4],
      ],
    };
    try {
      await sudokuCircuit.calculateWitness(input);
    } catch (err) {
      // console.log(err);
      assert(err.message.includes("Assert Failed"));
    }
  });
});
```

* To run tests using `yarn test` or `npm test` instead of `mocha test`, inside the `package.json` file add:
    

```json
"scripts": {
    "test": "mocha"
  },
```

The `package.json` file will look like this:

```json
{
  "name": "circuits",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "scripts": {
    "test": "mocha"
  },
  "dependencies": {
    "circom_tester": "^0.0.11",
    "circomlib": "^2.0.3"
  },
  "devDependencies": {
    "chai": "^4.3.6"
  }
}
```

* Run tests:
    

```bash
yarn test
```

You will see:

![CircuitsTestsImage.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654638667838/hYT5qq6R-.png align="left")

* If you uncomment the console logs, you will see that each test that should fail, is failing on the line of the circuit that should fail.
    

![CircuitsTestLogImage.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654659813001/QxfVYt3Cv.png align="left")

In this case it is line 114: `ieSquare[indexSquare].out === 0;`, when we check that elements are not equal to other elements in the same square.

## Smart Contracts

### 7\. Set up the local environment to work with smart contracts

For setting up the local environment to work with smart contracts, we are going to use the `hardhat` library. To know more about `hardhat`, read the [hardhat documentation](https://hardhat.org/getting-started/).

* Open a terminal inside the `zkSudoku` folder.
    
* Create the `contracts` folder:
    

```bash
mkdir contracts
```

* Go inside the `contracts` folder:
    

```bash
cd contracts
```

* Create the `package.json` file:
    

```bash
yarn init -y
```

* Add the `hardhat` library to work locally with smart contracts:
    

```bash
yarn add -D hardhat
```

* Create the hardhat project:
    

```bash
npx hardhat
```

* Select `Create a basic sample project` (it is the default option) and accept everything (press Enter).
    
* Compile and test smart contracts to make sure that everything is correct.
    
* To compile the contracts, run:
    

```bash
npx hardhat compile
```

* To test contracts, run:
    

```bash
npx hardhat test
```

* Open a code editor (inside the `zkSudoku/contracts` folder, in the hardhat project created).
    

**Note:** I use Visual Studio Code. To open Visual Studio Code, run:

```bash
code .
```

* Delete these files (**do not delete the folders**):
    
    * `sample-test.js` inside the `test` folder
        
    * `sample-script.js` inside the `scripts` folder
        
    * `Greeter.sol` inside the `contracts` folder
        

### 8\. Create smart contracts

* Copy the `sudokuVerifier.sol` generated before.
    

You can copy the file or you can run this:

```bash
cp ../circuits/sudoku/sudokuVerifier.sol contracts
```

* Inside the `zkSudoku/contracts/contracts` folder, create the `Sudoku.sol` file and add to it:
    

```solidity
//SPDX-License-Identifier: Unlicense
pragma solidity ^0.8.4;

interface IVerifier {
    function verifyProof(
        uint256[2] memory a,
        uint256[2][2] memory b,
        uint256[2] memory c,
        uint256[81] memory input
    ) external view returns (bool);
}

contract Sudoku {
    address public verifierAddr;

    uint8[9][9][3] sudokuBoardList = [
        [
            [1, 2, 7, 5, 8, 4, 6, 9, 3],
            [8, 5, 6, 3, 7, 9, 1, 2, 4],
            [3, 4, 9, 6, 2, 1, 8, 7, 5],
            [4, 7, 1, 9, 5, 8, 2, 3, 6],
            [2, 6, 8, 7, 1, 3, 5, 4, 9],
            [9, 3, 5, 4, 6, 2, 7, 1, 8],
            [5, 8, 3, 2, 9, 7, 4, 6, 1],
            [7, 1, 4, 8, 3, 6, 9, 5, 2],
            [6, 9, 2, 1, 4, 5, 3, 0, 7]
        ],
        [
            [0, 2, 7, 5, 0, 4, 0, 0, 0],
            [0, 0, 0, 3, 7, 0, 0, 0, 4],
            [3, 0, 0, 0, 0, 0, 8, 0, 0],
            [4, 7, 0, 9, 5, 8, 0, 3, 6],
            [2, 6, 8, 7, 1, 0, 0, 4, 9],
            [0, 0, 0, 0, 0, 2, 0, 1, 8],
            [0, 8, 3, 0, 9, 0, 4, 0, 0],
            [7, 1, 0, 0, 0, 0, 9, 0, 2],
            [0, 0, 0, 0, 0, 5, 0, 0, 7]
        ],
        [
            [0, 0, 0, 0, 0, 6, 0, 0, 0],
            [0, 0, 7, 2, 0, 0, 8, 0, 0],
            [9, 0, 6, 8, 0, 0, 0, 1, 0],
            [3, 0, 0, 7, 0, 0, 0, 2, 9],
            [0, 0, 0, 0, 0, 0, 0, 0, 0],
            [4, 0, 0, 5, 0, 0, 0, 7, 0],
            [6, 5, 0, 1, 0, 0, 0, 0, 0],
            [8, 0, 1, 0, 5, 0, 3, 0, 0],
            [7, 9, 2, 0, 0, 0, 0, 0, 4]
        ]
    ];

    constructor(address _verifierAddr) {
        verifierAddr = _verifierAddr;
    }

    function verifyProof(
        uint256[2] memory a,
        uint256[2][2] memory b,
        uint256[2] memory c,
        uint256[81] memory input
    ) public view returns (bool) {
        return IVerifier(verifierAddr).verifyProof(a, b, c, input);
    }

    function verifySudokuBoard(uint256[81] memory board)
        private
        view
        returns (bool)
    {
        bool isEqual = true;
        for (uint256 i = 0; i < sudokuBoardList.length; ++i) {
            isEqual = true;
            for (uint256 j = 0; j < sudokuBoardList[i].length; ++j) {
                for (uint256 k = 0; k < sudokuBoardList[i][j].length; ++k) {
                    if (board[9 * j + k] != sudokuBoardList[i][j][k]) {
                        isEqual = false;
                        break;
                    }
                }
            }
            if (isEqual == true) {
                return isEqual;
            }
        }
        return isEqual;
    }

    function verifySudoku(
        uint256[2] memory a,
        uint256[2][2] memory b,
        uint256[2] memory c,
        uint256[81] memory input
    ) public view returns (bool) {
        require(verifySudokuBoard(input), "This board does not exist");
        require(verifyProof(a, b, c, input), "Filed proof check");
        return true;
    }

    function pickRandomBoard(string memory stringTime)
        private
        view
        returns (uint8[9][9] memory)
    {
        uint256 randPosition = uint256(
            keccak256(
                abi.encodePacked(
                    block.difficulty,
                    block.timestamp,
                    msg.sender,
                    stringTime
                )
            )
        ) % sudokuBoardList.length;
        return sudokuBoardList[randPosition];
    }

    function generateSudokuBoard(string memory stringTime)
        public
        view
        returns (uint8[9][9] memory)
    {
        return pickRandomBoard(stringTime);
    }
}
```

* Smart contracts graph:
    

![SmartContractsGraph.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654660672261/zyfCWrdr8.png align="left")

### 9\. Test smart contracts

* Add the [snarkjs](https://github.com/iden3/snarkjs) library to test the generation of the proof:
    

```bash
yarn add snarkjs
```

**Note:** Make sure you have the same global and local version of `snarkjs`.

**1-** To check the global `snarkjs` version, open a console and run:

```bash
snarkjs -v
```

You will see something like this:

![globalsnarkjs.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1663532650986/ejzt5Y5PD.png align="left")

**2-** To check the local `snarkjs` version, go to the `package.json` file and check the `snarkjs` version there. You will see something like this:

![contractslocalsnarkjs.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1663535012941/-prtnTr9Q.png align="left")

You can see that both versions of `snarkjs` are the same: `0.4.19`.

* Create a `zkproof` folder:
    

```bash
mkdir zkproof
```

* Copy the `sudoku.wasm` and `sudoku_final.zkey` files inside the `zkproof` folder created before:
    
    * Copy the `sudoku.wasm` file inside the `zkproof` folder or run:
        
    
    ```bash
    cp ../circuits/sudoku/sudoku_js/sudoku.wasm zkproof
    ```
    
    * Copy the `sudoku_final.zkey` file inside the `zkproof` folder or run:
        
    
    ```bash
    cp ../circuits/sudoku/sudoku_final.zkey zkproof
    ```
    
* Add a `test.js` file inside the `test` folder and add to it:
    

```javascript
const { expect } = require("chai");
const { ethers } = require("hardhat");
const { exportCallDataGroth16 } = require("./utils/utils");

describe("Sudoku", function () {
  let SudokuVerifier, sudokuVerifier, Sudoku, sudoku;

  before(async function () {
    SudokuVerifier = await ethers.getContractFactory("SudokuVerifier");
    sudokuVerifier = await SudokuVerifier.deploy();
    await sudokuVerifier.deployed();

    Sudoku = await ethers.getContractFactory("Sudoku");
    sudoku = await Sudoku.deploy(sudokuVerifier.address);
    await sudoku.deployed();
  });

  it("Should generate a board", async function () {
    let board = await sudoku.generateSudokuBoard(new Date().toString());
    expect(board.length).to.equal(9);
  });

  it("Should return true for valid proof on-chain", async function () {
    const unsolved = [
      [1, 2, 7, 5, 8, 4, 6, 9, 3],
      [8, 5, 6, 3, 7, 9, 1, 2, 4],
      [3, 4, 9, 6, 2, 1, 8, 7, 5],
      [4, 7, 1, 9, 5, 8, 2, 3, 6],
      [2, 6, 8, 7, 1, 3, 5, 4, 9],
      [9, 3, 5, 4, 6, 2, 7, 1, 8],
      [5, 8, 3, 2, 9, 7, 4, 6, 1],
      [7, 1, 4, 8, 3, 6, 9, 5, 2],
      [6, 9, 2, 1, 4, 5, 3, 0, 7],
    ];

    const solved = [
      [1, 2, 7, 5, 8, 4, 6, 9, 3],
      [8, 5, 6, 3, 7, 9, 1, 2, 4],
      [3, 4, 9, 6, 2, 1, 8, 7, 5],
      [4, 7, 1, 9, 5, 8, 2, 3, 6],
      [2, 6, 8, 7, 1, 3, 5, 4, 9],
      [9, 3, 5, 4, 6, 2, 7, 1, 8],
      [5, 8, 3, 2, 9, 7, 4, 6, 1],
      [7, 1, 4, 8, 3, 6, 9, 5, 2],
      [6, 9, 2, 1, 4, 5, 3, 8, 7],
    ];

    const input = {
      unsolved: unsolved,
      solved: solved,
    };

    let dataResult = await exportCallDataGroth16(
      input,
      "./zkproof/sudoku.wasm",
      "./zkproof/sudoku_final.zkey"
    );

    // Call the function.
    let result = await sudokuVerifier.verifyProof(
      dataResult.a,
      dataResult.b,
      dataResult.c,
      dataResult.Input
    );
    expect(result).to.equal(true);
  });

  it("Should return false for invalid proof on-chain", async function () {
    let a = [0, 0];
    let b = [
      [0, 0],
      [0, 0],
    ];
    let c = [0, 0];
    let Input = new Array(81).fill(0);

    let dataResult = { a, b, c, Input };

    // Call the function.
    let result = await sudokuVerifier.verifyProof(
      dataResult.a,
      dataResult.b,
      dataResult.c,
      dataResult.Input
    );
    expect(result).to.equal(false);
  });
  it("Should verify Sudoku successfully", async function () {
    const unsolved = [
      [1, 2, 7, 5, 8, 4, 6, 9, 3],
      [8, 5, 6, 3, 7, 9, 1, 2, 4],
      [3, 4, 9, 6, 2, 1, 8, 7, 5],
      [4, 7, 1, 9, 5, 8, 2, 3, 6],
      [2, 6, 8, 7, 1, 3, 5, 4, 9],
      [9, 3, 5, 4, 6, 2, 7, 1, 8],
      [5, 8, 3, 2, 9, 7, 4, 6, 1],
      [7, 1, 4, 8, 3, 6, 9, 5, 2],
      [6, 9, 2, 1, 4, 5, 3, 0, 7],
    ];

    const solved = [
      [1, 2, 7, 5, 8, 4, 6, 9, 3],
      [8, 5, 6, 3, 7, 9, 1, 2, 4],
      [3, 4, 9, 6, 2, 1, 8, 7, 5],
      [4, 7, 1, 9, 5, 8, 2, 3, 6],
      [2, 6, 8, 7, 1, 3, 5, 4, 9],
      [9, 3, 5, 4, 6, 2, 7, 1, 8],
      [5, 8, 3, 2, 9, 7, 4, 6, 1],
      [7, 1, 4, 8, 3, 6, 9, 5, 2],
      [6, 9, 2, 1, 4, 5, 3, 8, 7],
    ];

    const input = {
      unsolved: unsolved,
      solved: solved,
    };

    let dataResult = await exportCallDataGroth16(
      input,
      "./zkproof/sudoku.wasm",
      "./zkproof/sudoku_final.zkey"
    );

    // Call the function.
    let result = await sudoku.verifySudoku(
      dataResult.a,
      dataResult.b,
      dataResult.c,
      dataResult.Input
    );
    expect(result).to.equal(true);
  });
  it("Should be reverted on Sudoku verification because the board is not in the board list", async function () {
    const unsolved = [
      [1, 2, 7, 5, 8, 4, 6, 9, 3],
      [8, 5, 6, 3, 7, 9, 1, 2, 4],
      [3, 4, 9, 6, 2, 1, 8, 7, 5],
      [4, 7, 1, 9, 5, 8, 2, 3, 6],
      [2, 6, 8, 7, 1, 3, 5, 4, 9],
      [9, 3, 5, 4, 6, 2, 7, 1, 8],
      [5, 8, 3, 2, 9, 7, 4, 6, 1],
      [7, 1, 4, 8, 3, 6, 9, 5, 2],
      [6, 9, 2, 1, 4, 5, 3, 8, 0],
    ];

    const solved = [
      [1, 2, 7, 5, 8, 4, 6, 9, 3],
      [8, 5, 6, 3, 7, 9, 1, 2, 4],
      [3, 4, 9, 6, 2, 1, 8, 7, 5],
      [4, 7, 1, 9, 5, 8, 2, 3, 6],
      [2, 6, 8, 7, 1, 3, 5, 4, 9],
      [9, 3, 5, 4, 6, 2, 7, 1, 8],
      [5, 8, 3, 2, 9, 7, 4, 6, 1],
      [7, 1, 4, 8, 3, 6, 9, 5, 2],
      [6, 9, 2, 1, 4, 5, 3, 8, 7],
    ];

    const input = {
      unsolved: unsolved,
      solved: solved,
    };

    let dataResult = await exportCallDataGroth16(
      input,
      "./zkproof/sudoku.wasm",
      "./zkproof/sudoku_final.zkey"
    );

    await expect(
      sudoku.verifySudoku(
        dataResult.a,
        dataResult.b,
        dataResult.c,
        dataResult.Input
      )
    ).to.be.reverted;
  });
});
```

* Inside a `test` folder, create a `utils` folder.
    
* Inside the `utils` folder, create a `utils.js` file and add to it:
    

```javascript
const { groth16 } = require("snarkjs");

async function exportCallDataGroth16(input, wasmPath, zkeyPath) {
  const { proof: _proof, publicSignals: _publicSignals } =
    await groth16.fullProve(input, wasmPath, zkeyPath);
  const calldata = await groth16.exportSolidityCallData(_proof, _publicSignals);

  const argv = calldata
    .replace(/["[\]\s]/g, "")
    .split(",")
    .map((x) => BigInt(x).toString());

  const a = [argv[0], argv[1]];
  const b = [
    [argv[2], argv[3]],
    [argv[4], argv[5]],
  ];
  const c = [argv[6], argv[7]];
  const Input = [];

  for (let i = 8; i < argv.length; i++) {
    Input.push(argv[i]);
  }

  return { a, b, c, Input };
}

module.exports = {
  exportCallDataGroth16,
};
```

* To have the gas reporter when testing smart contracts, install the [hardhat-gas-reporter library](https://github.com/cgewecke/hardhat-gas-reporter). Run:
    

```bash
yarn add -D hardhat-gas-reporter
```

Then, in the `hardhat.config.js` file, after the last `require`, at the top, add:

```javascript
require("hardhat-gas-reporter");
```

* To add the optimizer, modify your solidity config in the `hardhat.config.js` file like this:
    

```javascript
solidity: {
    version: "0.8.4",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200,
      },
    },
  },
```

* To test contracts, run:
    

```bash
npx hardhat test
```

When you run the above line, you will see:

![RunTestsImage.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654638709579/h8jb_erNp.png align="left")

### 10\. Run smart contracts

* Inside the `scripts` folder, add a `run.js` file (to play around with smart contracts) and add to it:
    

```javascript
const main = async () => {
  const SudokuVerifier = await hre.ethers.getContractFactory("SudokuVerifier");
  const sudokuVerifier = await SudokuVerifier.deploy();
  await sudokuVerifier.deployed();
  console.log("SudokuVerifier Contract deployed to:", sudokuVerifier.address);

  const Sudoku = await hre.ethers.getContractFactory("Sudoku");
  const sudoku = await Sudoku.deploy(sudokuVerifier.address);
  await sudoku.deployed();
  console.log("Sudoku Contract deployed to:", sudoku.address);

  let board = await sudoku.generateSudokuBoard(new Date().toString());
  console.log(board);

  let callDataSudoku = [
    [
      "0x2c5defbc1706b51a941bff57cc1e40f2c941fccbbf13c587de8bf36dc1017b56",
      "0x10c1184d623f5f8efa15052e3c59613a80507c11f0605c998c36a0336bb4012f",
    ],
    [
      [
        "0x21fcd189100631f163579f405875def48a41c0f6b1dae436a71feebcc84af110",
        "0x0ac4517b8693891029159dd4d5b15e17fbe0ac27068cf14a9781e3a347352cd6",
      ],
      [
        "0x0f43d6e3ace2228a67578fcb7778d248b274f6342ed14828454d343c0a933110",
        "0x075df9f66bb65d47e04caa80d6f8e99fbd8db5217d259a3aac80709b5cb37347",
      ],
    ],
    [
      "0x05f56ad16658e304882d42230c634626848f981c51d7a34f5f9833f7272447fb",
      "0x1405302f7a377aa913513f026ea72e3544484d66ce27204e72787ce4bf3229b4",
    ],
    [
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000006",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000007",
      "0x0000000000000000000000000000000000000000000000000000000000000002",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000008",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000009",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000006",
      "0x0000000000000000000000000000000000000000000000000000000000000008",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000001",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000003",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000007",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000002",
      "0x0000000000000000000000000000000000000000000000000000000000000009",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000004",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000005",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000007",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000006",
      "0x0000000000000000000000000000000000000000000000000000000000000005",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000001",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000008",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000001",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000005",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000003",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000007",
      "0x0000000000000000000000000000000000000000000000000000000000000009",
      "0x0000000000000000000000000000000000000000000000000000000000000002",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000004",
    ],
  ];

  // Call the function.
  let result = await sudokuVerifier.verifyProof(
    callDataSudoku[0],
    callDataSudoku[1],
    callDataSudoku[2],
    callDataSudoku[3]
  );

  console.log("Result", result);
};

const runMain = async () => {
  try {
    await main();
    process.exit(0);
  } catch (error) {
    console.log(error);
    process.exit(1);
  }
};

runMain();
```

* `callDataSudoku` is the data of the `parameters.txt` file generated before, the `verifyProof` function will return true. If you change an element of the `callDataSudoku` variable, the `verifyProof` function will return false.
    
* Run the `run.js` file:
    

```bash
npx hardhat run scripts/run.js
```

You will see something like this:

![RunContractsImage.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654638733833/iikcFrIPv.png align="left")

### 11\. Deploy smart contracts

We are going to deploy smart contracts on [Sepolia](https://sepolia.etherscan.io/).

We will use [Metamask](https://metamask.io/).

**Note:** You can add the Sepolia network on Metamask by following the [Sepolia website](https://sepolia.dev/) by clicking the `Add to Metamask` button.

* Inside the `scripts` folder, add a `deploy.js` file and add to it:
    

```javascript
const main = async () => {
  const SudokuVerifier = await hre.ethers.getContractFactory("SudokuVerifier");
  const sudokuVerifier = await SudokuVerifier.deploy();
  await sudokuVerifier.deployed();
  console.log("SudokuVerifier Contract deployed to:", sudokuVerifier.address);

  const Sudoku = await hre.ethers.getContractFactory("Sudoku");
  const sudoku = await Sudoku.deploy(sudokuVerifier.address);
  await sudoku.deployed();
  console.log("Sudoku Contract deployed to:", sudoku.address);
};

const runMain = async () => {
  try {
    await main();
    process.exit(0);
  } catch (error) {
    console.log(error);
    process.exit(1);
  }
};

runMain();
```

**Note:** It is almost the same as `run.js` but only deployed smart contracts.

* Deploy smart contracts on Sepolia.
    
* Create a `.env` file and add to it:
    

```bash
PRIVATE_KEY=
```

**Note:** After the `=` symbol, add the private key of your wallet. Check that the `.env` file is in the `.gitignore` file like this:

```bash
node_modules
.env
coverage
coverage.json
typechain

#Hardhat files
cache
artifacts
```

* Install the [dotenv library](https://github.com/motdotla/dotenv):
    

```bash
yarn add dotenv
```

* Go to the `hardhat.config.js` file and edit `module.exports` like this:
    

```javascript
module.exports = {
  solidity: {
    version: "0.8.4",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200,
      },
    },
  },
  networks: {
    sepolia: {
      url: "https://rpc.sepolia.org/",
      accounts: [process.env.PRIVATE_KEY],
    },
    mumbai: {
      url: "https://rpc-mumbai.maticvigil.com",
      accounts: [process.env.PRIVATE_KEY],
    },
  },
};
```

* Go to `hardhat.config.js` and add after the last `require` at the top:
    

```javascript
require("dotenv").config();
```

* Your `hardhat.config.js` file should look like this:
    

```javascript
require("@nomiclabs/hardhat-waffle");
require("hardhat-gas-reporter");
require("dotenv").config();

// This is a sample Hardhat task. To learn how to create your own go to
// https://hardhat.org/guides/create-task.html
task("accounts", "Prints the list of accounts", async (taskArgs, hre) => {
  const accounts = await hre.ethers.getSigners();

  for (const account of accounts) {
    console.log(account.address);
  }
});

// You need to export an object to set up your config
// Go to https://hardhat.org/config/ to learn more

/**
 * @type import('hardhat/config').HardhatUserConfig
 */
module.exports = {
  solidity: {
    version: "0.8.4",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200,
      },
    },
  },
  networks: {
    sepolia: {
      url: "https://rpc.sepolia.org/",
      accounts: [process.env.PRIVATE_KEY],
    },
    mumbai: {
      url: "https://rpc-mumbai.maticvigil.com",
      accounts: [process.env.PRIVATE_KEY],
    },
  },
};
```

* Get Sepolia ETH faucet for Testnet:
    

Go to [https://sepoliafaucet.com/](https://sepoliafaucet.com/) and follow the instructions there.

* Run the `deploy.js` file:
    

```bash
npx hardhat run scripts/deploy.js --network sepolia
```

![DeployedContractsImage.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654638753683/tf1mjf8Pa.png align="left")

**Note:** You can see the transactions on the Sepolia Block Explorer: [https://sepolia.etherscan.io/](https://sepolia.etherscan.io/)

## Frontend

### 12\. Creating the application

* Create the [Next.js Application](https://nextjs.org/).
    

Open a terminal inside the `zkSudoku` folder and run:

```bash
yarn create next-app zksudoku-ui
```

* Go inside the `zksudoku-ui` folder:
    

```bash
cd zksudoku-ui
```

* Open a code editor inside the `zksudoku-ui` folder.
    

**Note:** I use Visual Studio Code. To open Visual Studio Code, run:

```bash
code .
```

* To test that everything is fine let's start the server, run:
    

```bash
yarn dev
```

Open a browser and go to [http://localhost:3000/](http://localhost:3000/).

You will see this:

![NextjsInitialPageImage.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654638771868/CXzQZBQ4z.png align="left")

* Stop the server and let's start building the game.
    

### 13\. Add some libraries

* Add [Tailwind](https://tailwindcss.com/) to style pages.
    

Follow the [oficial documentation for adding Tailwind in Next.js](https://tailwindcss.com/docs/guides/nextjs).

* Add [wagmi](https://wagmi.sh/) to comunicate with smart contracts in the frontend:
    

```bash
yarn add wagmi ethers
```

* Add [snarkjs](https://github.com/iden3/snarkjs) to generate zk proof in the browser:
    

```bash
yarn add snarkjs
```

**Note:** Make sure you have the same global and local version of `snarkjs`.

**1-** To check the global `snarkjs` version, open a console and run:

```bash
snarkjs -v
```

You will see something like this:

![globalsnarkjs.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1663532650986/ejzt5Y5PD.png align="left")

**2-** To check the local `snarkjs` version, go to the `package.json` file and check the `snarkjs` version there. You will see something like this:

![localsnarkjs.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1663532693336/9VMT2vbmT.png align="left")

You can see that both versions of `snarkjs` are the same: `0.4.19`.

Two ways you can import `snarkjs` in the client side:

**1-** Using the `snarkjs.min.js` file.

Copy the `snarkjs.min.js` inside the `public` folder:

```bash
cp ./node_modules/snarkjs/build/snarkjs.min.js ./public
```

Add `<Script id="snarkjs" src="/snarkjs.min.js" />` in `layout.js`

You can access the library using `window.snarkjs`.

**2-** As a package.

To install `snarkjs`, run:

```bash
yarn add snarkjs
```

Configure webpack in the `next.config.js` file to use `snarkjs`:

```javascript
const nextConfig = {
  reactStrictMode: true,
  webpack: function (config, options) {
    if (!options.isServer) {
      config.resolve.fallback.fs = false;
    }
    config.experiments = { asyncWebAssembly: true };
    return config;
  },
};

module.exports = nextConfig;
```

**Note:** The `config.experiments = { asyncWebAssembly: true };` line is for using wasm files.

In this guide, I use `snarkjs` as a package (way 2).

### 14\. Add some configuration

* To generate the proof, add a `.babelrc` file and add to it:
    

```javascript
{
  "presets": [
    [
      "next/babel",
      {
        "preset-env": {
          // "debug": true,
          "targets": [
            "last 2 Edge versions",
            "last 2 Opera versions",
            "last 2 Safari versions",
            "last 2 Chrome versions",
            "last 2 Firefox versions"
          ]
        }
      }
    ]
  ]
}
```

### 15\. Add Zero Knowledge in the frontend

* Inside the `public` folder, add a `zkproof` folder and copy the `sudoku.wasm` and `sudoku_final.key` files there:
    
    * To copy the `sudoku.wasm` file:
        
    
    ```bash
    cp ../../circuits/sudoku/sudoku_js/sudoku.wasm ./public/zkproof
    ```
    
    * To copy the `sudoku_final.zkey` file:
        
    
    ```bash
    cp ../../circuits/sudoku/sudoku_final.zkey ./public/zkproof
    ```
    
* Inside the `zksudoku-ui` folder, create a `zkproof` folder. Inside this new folder created, create the `snarkjsZkproof.js` file and add to it:
    

```javascript
import { groth16 } from "snarkjs";

export async function exportCallDataGroth16(input, wasmPath, zkeyPath) {
  const { proof: _proof, publicSignals: _publicSignals } =
    await groth16.fullProve(input, wasmPath, zkeyPath);

  const calldata = await groth16.exportSolidityCallData(_proof, _publicSignals);

  const argv = calldata
    .replace(/["[\]\s]/g, "")
    .split(",")
    .map((x) => BigInt(x).toString());

  const a = [argv[0], argv[1]];
  const b = [
    [argv[2], argv[3]],
    [argv[4], argv[5]],
  ];
  const c = [argv[6], argv[7]];
  const Input = [];

  for (let i = 8; i < argv.length; i++) {
    Input.push(argv[i]);
  }

  return { a, b, c, Input };
}
```

**Note:** If you get an error importing `groth16` from `snarkjs` that says:

```bash
./node_modules/fastfile/src/fastfile.js
Can't import the named export 'O_TRUNC' (imported as 'O_TRUNC') from default-exporting module (only default export is available)
```

Then, import `groth16` like so:

```javascript
const groth16 = require("snarkjs").groth16;
```

Or this way:

```javascript
const { groth16 } = require("snarkjs");
```

**Note:** If you get an error importing `groth16` from `snarkjs` that says:

```bash
./node_modules/snarkjs/build/main.cjs:8:0
Module not found: Can't resolve 'readline'
```

Then, add the `config.resolve.fallback.readline = false;` line in the `next.config.js` file like so:

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  webpack: function (config, options) {
    if (!options.isServer) {
      config.resolve.fallback.fs = false;
      config.resolve.fallback.readline = false;
    }
    config.experiments = { asyncWebAssembly: true };
    return config;
  },
};

module.exports = nextConfig;
```

* Inside the `zksudoku-ui/zkproof` folder, create the `sudoku` folder and inside this new folder created, create the `snarkjsSudoku.js` file and add to it:
    

```javascript
import { exportCallDataGroth16 } from "../snarkjsZkproof";

export async function sudokuCalldata(unsolved, solved) {
  const input = {
    unsolved: unsolved,
    solved: solved,
  };

  let dataResult;

  try {
    dataResult = await exportCallDataGroth16(
      input,
      "/zkproof/sudoku.wasm",
      "/zkproof/sudoku_final.zkey"
    );
  } catch (error) {
    // console.log(error);
    window.alert("Wrong answer");
  }

  return dataResult;
}
```

### 16\. Connect to smart contracts to verify the proof

* Code to connect to smart contracts:
    

```javascript
const contract = useContract({
  addressOrName: contractAddress.sudokuContract,
  contractInterface: sudokuContractAbi.abi,
  signerOrProvider: signer || provider,
});
```

* Code to use the `verifySudoku` function which is in the `Sudoku` smart contract:
    

```javascript
result = await contract.verifySudoku(
  calldata.a,
  calldata.b,
  calldata.c,
  calldata.Input
);
```

**Note:** The two code blocks above are inside `pages/sudoku.js`.

### 17\. Add utility files

* Inside the `zksudoku-ui` folder add the `utils` folder.
    
* Inside the `utils` folder created, add:
    
    * The `abiFiles` folder which contains all the abi files needed for the frontend application. You can find the abi file here: `zkSudoku/contracts/artifacts/contracts/Sudoku.sol/Sudoku.json`. The abi file is a file generated when the smart contract is compiled.
        
    * The `contractsaddress.json` file that contains all the necessary smart contracts addresses, in this case, the `Sudoku` contract:
        
    
    ```json
    {
      "sudokuContract": "0xCdf6cb8d73A7D382f154A1e67F12c7319987cb31"
    }
    ```
    
    * The `networks.json` file which contains all the chains that we can use in the app and the `selectedChain` which is the network used in this project (Sepolia):
        
    
    ```json
    {
      "selectedChain": "11155111",
      "1337": {
        "chainId": "1337",
        "chainName": "Localhost 8545",
        "rpcUrls": ["http://localhost:8545"],
        "nativeCurrency": {
          "symbol": "ETH"
        },
        "blockExplorerUrls": []
      },
      "11155111": {
        "chainId": "11155111",
        "chainName": "Sepolia",
        "rpcUrls": ["https://rpc.sepolia.org"],
        "nativeCurrency": {
          "symbol": "ETH"
        },
        "blockExplorerUrls": ["https://sepolia.etherscan.io/"]
      },
      "80001": {
        "chainId": "80001",
        "chainName": "Mumbai",
        "rpcUrls": ["https://rpc-mumbai.maticvigil.com"],
        "nativeCurrency": {
          "symbol": "MATIC"
        },
        "blockExplorerUrls": ["https://mumbai.polygonscan.com/"]
      }
    }
    ```
    
    * The `switchNetwork.js` file to switch to the network used in the projects if necessary. (In this project we are using Sepolia)
        
    
    ```javascript
    import networks from "../utils/networks.json";
    
    export const switchNetwork = async () => {
      if (window.ethereum) {
        try {
          // Try to switch to the chain
          await ethereum.request({
            method: "wallet_switchEthereumChain",
            params: [
              { chainId: `0x${parseInt(networks.selectedChain).toString(16)}` },
            ],
          });
        } catch (switchError) {
          // This error code indicates that the chain has not been added to MetaMask.
          if (switchError.code === 4902) {
            try {
              await ethereum.request({
                method: "wallet_addEthereumChain",
                params: [
                  {
                    chainId: `0x${parseInt(networks.selectedChain).toString(16)}`,
                    chainName: networks[networks.selectedChain].chainName,
                    rpcUrls: networks[networks.selectedChain].rpcUrls,
                    nativeCurrency: {
                      symbol:
                        networks[networks.selectedChain].nativeCurrency.symbol,
                      decimals: 18,
                    },
                    blockExplorerUrls:
                      networks[networks.selectedChain].blockExplorerUrls,
                  },
                ],
              });
            } catch (addError) {
              console.log(addError);
            }
          }
          // handle other "switch" errors
        }
      } else {
        // If window.ethereum is not found then MetaMask is not installed
        alert(
          "MetaMask is not installed. Please install it to use this app: https://metamask.io/download/"
        );
      }
    };
    ```
    

### 18\. Add other files

* Delete:
    
    * The `api` folder inside the `pages` folder.
        
    * The `favicon.ico` file, inside the `public` folder.
        
    * The `vercel.svg` file, inside the `public` folder.
        
* Create:
    
    * The `assets` folder and add the image inside it.
        
    * The `components` folder and copy all the code inside that folder.
        
* Copy:
    
    * All the files inside the `pages` folder.
        
    * All the files inside the `styles` folder.
        
    * The `favicon.ico` and `socialMedia.png` files, inside the `public` folder.
        

### 19\. Deploy the frontend

We are going to deploy the frontend on [Vercel](https://vercel.com/).

* Create a new [Github](https://github.com/) repository.
    
* Go to Vercel:
    
    * Create a new project.
        
    * Import the Github repo created before.
        
    * Configure the project (select `Next.js` as FRAMEWORK PRESET and `zksudoku-ui` as ROOT DIRECTORY) and click `Deploy`:
        
    
    ![VercelConfigImage.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654643769359/823-B6u_6.png align="left")
    

#### Live app:

[https://zk-sudoku.vercel.app/](https://zk-sudoku.vercel.app/)

#### Live App Images:

* Initial Page:
    

![LiveAppImage.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654644050000/xDHVwf0jX.png align="left")

* Wrong answer:
    

![LiveAppWrongAnswerImage.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654644110903/g9QMcDnhV.png align="left")

* Successfully verified:
    

![LiveAppSuccessfullyVerifiedImage.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654644173085/B5fLQ9DFs.png align="left")

## Some Tips

* If you change the circuit, you should update:
    
    * `sudokuVerifier.sol` inside `contracts/contracts`.
        
    * `sudoku.wasm` and `sudoku_final/zkey` files inside `contracts/zkproof`.
        
    * `sudoku.wasm` and `sudoku_final/zkey` files inside `zksudoku-ui/public/zkproof`.
        
* If you change smart contracts you should update if necessary:
    
    * The `Sudoku.json` abi file inside `zkSudoku-ui/utils/abiFiles`.
        
    * The `contractaddress.json` file inside `zkSudoku-ui/utils/`(with the new Sudoku smart contract address in case smart contracts are deployed again).
        
* If you change the network used to deploy you should update `selectedChain` inside `zkSudoku-ui/utils/networks.json`.
    
* To copy or update all the zk elements to use and test smart contracts, you can create the `copyZkFiles.sh` file inside `zkSudoku/contracts/contracts/scripts` and add to it:
    

```bash
#!/bin/bash

# Copy the verifier
cp ../circuits/sudoku/sudokuVerifier.sol contracts

# Create the zkproof folder if it does not exist
mkdir -p zkproof

# Copy the wasm file to test smart contracts
cp ../circuits/sudoku/sudoku_js/sudoku.wasm zkproof

# Copy the final zkey file to test smart contracts
cp ../circuits/sudoku/sudoku_final.zkey zkproof
```

Run the first time:

```bash
chmod u+x scripts/copyZkFiles.sh
```

And after that, you can always run:

```bash
./scripts/copyZkFiles.sh
```

* To copy or update all the zk elements to use in the frontend, you can create the `copyZkFiles.sh` file inside `zkSudoku/zkSudoku-ui/scripts` and add to it:
    

```bash
#!/bin/bash

# Create the zkproof folder inside the public folder if it does not exist
mkdir -p public/zkproof

# Copy the wasm file
cp ../circuits/sudoku/sudoku_js/sudoku.wasm public/zkproof

# Copy the final zkey
cp ../circuits/sudoku/sudoku_final.zkey public/zkproof

# Create the abiFiles folder inside the utils folder if it does not exist
mkdir -p utils/abiFiles

# Copy the abi file
cp ../contracts/artifacts/contracts/Sudoku.sol/Sudoku.json utils/abiFiles
```

Run the first time:

```bash
chmod u+x scripts/copyZkFiles.sh
```

And after that, you can always run:

```bash
./scripts/copyZkFiles.sh
```

## Create a zk dapp using Plonk instead of Groth16

To create a zk dapp using Plonk instead of Groth16, you can follow the same steps before but changing some files in some steps.

### Circuits changes

* We are not going to use Groth16, so instead of `executeGoth16.sh`, let's add a `executePlonk.sh` file and add to it:
    

```bash
#!/bin/bash

# Variable to store the name of the circuit
CIRCUIT=sudoku

# Variable to store the number of the ptau file
PTAU=15

# In case there is a circuit name as an input
if [ "$1" ]; then
    CIRCUIT=$1
fi

# In case there is a ptau file number as an input
if [ "$2" ]; then
    PTAU=$2
fi

# Check if the necessary ptau file already exists. If it does not exist, it will be downloaded from the data center
if [ -f ./ptau/powersOfTau28_hez_final_${PTAU}.ptau ]; then
    echo "----- powersOfTau28_hez_final_${PTAU}.ptau already exists -----"
else
    echo "----- Download powersOfTau28_hez_final_${PTAU}.ptau -----"
    wget -P ./ptau https://hermez.s3-eu-west-1.amazonaws.com/powersOfTau28_hez_final_${PTAU}.ptau
fi

# Compile the circuit
circom ${CIRCUIT}.circom --r1cs --wasm --sym --c

# Generate the witness.wtns
node ${CIRCUIT}_js/generate_witness.js ${CIRCUIT}_js/${CIRCUIT}.wasm input.json ${CIRCUIT}_js/witness.wtns

echo "----- Generate .zkey file -----"
# Generate a .zkey file that will contain the proving and verification keys together with all phase 2 contributions
snarkjs plonk setup ${CIRCUIT}.r1cs ptau/powersOfTau28_hez_final_${PTAU}.ptau ${CIRCUIT}_final.zkey

echo "----- Export the verification key -----"
# Export the verification key
snarkjs zkey export verificationkey ${CIRCUIT}_final.zkey verification_key.json

echo "----- Generate zk-proof -----"
# Generate a zk-proof associated to the circuit and the witness. This generates proof.json and public.json
snarkjs plonk prove ${CIRCUIT}_final.zkey ${CIRCUIT}_js/witness.wtns proof.json public.json

echo "----- Verify the proof -----"
# Verify the proof
snarkjs plonk verify verification_key.json public.json proof.json

echo "----- Generate Solidity verifier -----"
# Generate a Solidity verifier that allows verifying proofs on Ethereum blockchain
snarkjs zkey export solidityverifier ${CIRCUIT}_final.zkey ${CIRCUIT}PlonkVerifier.sol
# Update the solidity version in the Solidity verifier
sed -i "s/>=0.7.0 <0.9.0;/^0.8.4;/g" ${CIRCUIT}PlonkVerifier.sol
# Update the contract name in the Solidity verifier
sed -i "s/contract PlonkVerifier/contract SudokuPlonkVerifier/g" ${CIRCUIT}PlonkVerifier.sol

echo "----- Generate and print parameters of call -----"
# Generate and print parameters of call
snarkjs generatecall | tee parameters.txt
```

When you run the above `executePlonk.sh` file you will see:

![ExecutePlonkImage.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654638792082/DIcg5jQ41.png align="left")

Here we needed the `powersOfTau28_hez_final_15.ptau` instead of `powersOfTau28_hez_final_14.ptau` (as we used with Groth16) because the amount of Plonk constraints is 17901 and it is &gt; 2\*\*14.

![PlonkPowersOfTauImage.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654638814320/SHjRDcQ3L.png align="left")

### Smart Contracts changes

* The `Sudoku.sol` file changes a little because the `IVerifier` interface uses different input because of `verifyProof(bytes memory proof, uint[] memory pubSignals)` inside the generated `sudokuPlonkVerifier.sol` :
    

```solidity
//SPDX-License-Identifier: Unlicense
pragma solidity ^0.8.4;

interface IVerifier {
    function verifyProof(bytes memory proof, uint256[] memory pubSignals)
        external
        view
        returns (bool);
}

contract Sudoku {
    address public verifierAddr;

    uint8[9][9][3] sudokuBoardList = [
        [
            [1, 2, 7, 5, 8, 4, 6, 9, 3],
            [8, 5, 6, 3, 7, 9, 1, 2, 4],
            [3, 4, 9, 6, 2, 1, 8, 7, 5],
            [4, 7, 1, 9, 5, 8, 2, 3, 6],
            [2, 6, 8, 7, 1, 3, 5, 4, 9],
            [9, 3, 5, 4, 6, 2, 7, 1, 8],
            [5, 8, 3, 2, 9, 7, 4, 6, 1],
            [7, 1, 4, 8, 3, 6, 9, 5, 2],
            [6, 9, 2, 1, 4, 5, 3, 0, 7]
        ],
        [
            [0, 2, 7, 5, 0, 4, 0, 0, 0],
            [0, 0, 0, 3, 7, 0, 0, 0, 4],
            [3, 0, 0, 0, 0, 0, 8, 0, 0],
            [4, 7, 0, 9, 5, 8, 0, 3, 6],
            [2, 6, 8, 7, 1, 0, 0, 4, 9],
            [0, 0, 0, 0, 0, 2, 0, 1, 8],
            [0, 8, 3, 0, 9, 0, 4, 0, 0],
            [7, 1, 0, 0, 0, 0, 9, 0, 2],
            [0, 0, 0, 0, 0, 5, 0, 0, 7]
        ],
        [
            [0, 0, 0, 0, 0, 6, 0, 0, 0],
            [0, 0, 7, 2, 0, 0, 8, 0, 0],
            [9, 0, 6, 8, 0, 0, 0, 1, 0],
            [3, 0, 0, 7, 0, 0, 0, 2, 9],
            [0, 0, 0, 0, 0, 0, 0, 0, 0],
            [4, 0, 0, 5, 0, 0, 0, 7, 0],
            [6, 5, 0, 1, 0, 0, 0, 0, 0],
            [8, 0, 1, 0, 5, 0, 3, 0, 0],
            [7, 9, 2, 0, 0, 0, 0, 0, 4]
        ]
    ];

    constructor(address _verifierAddr) {
        verifierAddr = _verifierAddr;
    }

    function verifyProof(bytes memory proof, uint256[] memory pubSignals)
        public
        view
        returns (bool)
    {
        return IVerifier(verifierAddr).verifyProof(proof, pubSignals);
    }

    function verifySudokuBoard(uint256[] memory board)
        private
        view
        returns (bool)
    {
        bool isEqual = true;
        for (uint256 i = 0; i < sudokuBoardList.length; ++i) {
            isEqual = true;
            for (uint256 j = 0; j < sudokuBoardList[i].length; ++j) {
                for (uint256 k = 0; k < sudokuBoardList[i][j].length; ++k) {
                    if (board[9 * j + k] != sudokuBoardList[i][j][k]) {
                        isEqual = false;
                        break;
                    }
                }
            }
            if (isEqual == true) {
                return isEqual;
            }
        }
        return isEqual;
    }

    function verifySudoku(bytes memory proof, uint256[] memory pubSignals)
        public
        view
        returns (bool)
    {
        require(verifySudokuBoard(pubSignals), "This board does not exist");
        require(verifyProof(proof, pubSignals), "Filed proof check");
        return true;
    }

    function pickRandomBoard(string memory stringTime)
        private
        view
        returns (uint8[9][9] memory)
    {
        uint256 randPosition = uint256(
            keccak256(
                abi.encodePacked(
                    block.difficulty,
                    block.timestamp,
                    msg.sender,
                    stringTime
                )
            )
        ) % sudokuBoardList.length;
        return sudokuBoardList[randPosition];
    }

    function generateSudokuBoard(string memory stringTime)
        public
        view
        returns (uint8[9][9] memory)
    {
        return pickRandomBoard(stringTime);
    }
}
```

* The `copyZkFiles.sh` changes because the verifier was called `sudokuPlonkVerifier`:
    

```bash
#!/bin/bash

# Copy the verifier
cp ../circuits/sudoku/sudokuPlonkVerifier.sol contracts

# Create the zkproof folder if it does not exist
mkdir -p zkproof

# Copy the wasm file to test smart contracts
cp ../circuits/sudoku/sudoku_js/sudoku.wasm zkproof

# Copy the final zkey file to test smart contracts
cp ../circuits/sudoku/sudoku_final.zkey zkproof
```

* Change the `utils.js` file inside `test/utils` folder:
    

```javascript
const { plonk } = require("snarkjs");

async function exportCallDataPlonk(input, wasmPath, zkeyPath) {
  const { proof: _proof, publicSignals: _publicSignals } =
    await plonk.fullProve(input, wasmPath, zkeyPath);
  const calldata = await plonk.exportSolidityCallData(_proof, _publicSignals);

  // console.log("calldata", calldata);
  const calldataSplit = calldata.split(",");
  const [proof, ...rest] = calldataSplit;
  const publicSignals = JSON.parse(rest.join(",")).map((x) =>
    BigInt(x).toString()
  );
  return { proof, publicSignals };
}

module.exports = {
  exportCallDataPlonk,
};
```

* Change the `test.js` file inside the `test` folder:
    

```javascript
const { expect } = require("chai");
const { ethers } = require("hardhat");
const { exportCallDataPlonk } = require("./utils/utils");

describe("Sudoku", function () {
  let SudokuPlonkVerifier, sudokuPlonkVerifier, Sudoku, sudoku;

  before(async function () {
    SudokuPlonkVerifier = await ethers.getContractFactory(
      "SudokuPlonkVerifier"
    );
    sudokuPlonkVerifier = await SudokuPlonkVerifier.deploy();
    await sudokuPlonkVerifier.deployed();

    Sudoku = await ethers.getContractFactory("Sudoku");
    sudoku = await Sudoku.deploy(sudokuPlonkVerifier.address);
    await sudoku.deployed();
  });

  it("Should generate a board", async function () {
    let board = await sudoku.generateSudokuBoard(new Date().toString());
    expect(board.length).to.equal(9);
  });

  it("Should return true for valid proof on-chain", async function () {
    this.timeout(50000);
    const unsolved = [
      [1, 2, 7, 5, 8, 4, 6, 9, 3],
      [8, 5, 6, 3, 7, 9, 1, 2, 4],
      [3, 4, 9, 6, 2, 1, 8, 7, 5],
      [4, 7, 1, 9, 5, 8, 2, 3, 6],
      [2, 6, 8, 7, 1, 3, 5, 4, 9],
      [9, 3, 5, 4, 6, 2, 7, 1, 8],
      [5, 8, 3, 2, 9, 7, 4, 6, 1],
      [7, 1, 4, 8, 3, 6, 9, 5, 2],
      [6, 9, 2, 1, 4, 5, 3, 0, 7],
    ];

    const solved = [
      [1, 2, 7, 5, 8, 4, 6, 9, 3],
      [8, 5, 6, 3, 7, 9, 1, 2, 4],
      [3, 4, 9, 6, 2, 1, 8, 7, 5],
      [4, 7, 1, 9, 5, 8, 2, 3, 6],
      [2, 6, 8, 7, 1, 3, 5, 4, 9],
      [9, 3, 5, 4, 6, 2, 7, 1, 8],
      [5, 8, 3, 2, 9, 7, 4, 6, 1],
      [7, 1, 4, 8, 3, 6, 9, 5, 2],
      [6, 9, 2, 1, 4, 5, 3, 8, 7],
    ];

    const input = {
      unsolved: unsolved,
      solved: solved,
    };

    let dataResult = await exportCallDataPlonk(
      input,
      "./zkproof/sudoku.wasm",
      "./zkproof/sudoku_final.zkey"
    );

    // Call the function.
    let result = await sudokuPlonkVerifier.verifyProof(
      dataResult.proof,
      dataResult.publicSignals
    );
    expect(result).to.equal(true);
  });

  it("Should return false for invalid proof on-chain", async function () {
    let proof = 0;
    let publicSignals = new Array(81).fill(0);

    let dataResult = { proof, publicSignals };

    // Call the function.
    let result = await sudokuPlonkVerifier.verifyProof(
      dataResult.proof,
      dataResult.publicSignals
    );
    expect(result).to.equal(false);
  });
  it("Should verify Sudoku successfully", async function () {
    this.timeout(50000);
    const unsolved = [
      [1, 2, 7, 5, 8, 4, 6, 9, 3],
      [8, 5, 6, 3, 7, 9, 1, 2, 4],
      [3, 4, 9, 6, 2, 1, 8, 7, 5],
      [4, 7, 1, 9, 5, 8, 2, 3, 6],
      [2, 6, 8, 7, 1, 3, 5, 4, 9],
      [9, 3, 5, 4, 6, 2, 7, 1, 8],
      [5, 8, 3, 2, 9, 7, 4, 6, 1],
      [7, 1, 4, 8, 3, 6, 9, 5, 2],
      [6, 9, 2, 1, 4, 5, 3, 0, 7],
    ];

    const solved = [
      [1, 2, 7, 5, 8, 4, 6, 9, 3],
      [8, 5, 6, 3, 7, 9, 1, 2, 4],
      [3, 4, 9, 6, 2, 1, 8, 7, 5],
      [4, 7, 1, 9, 5, 8, 2, 3, 6],
      [2, 6, 8, 7, 1, 3, 5, 4, 9],
      [9, 3, 5, 4, 6, 2, 7, 1, 8],
      [5, 8, 3, 2, 9, 7, 4, 6, 1],
      [7, 1, 4, 8, 3, 6, 9, 5, 2],
      [6, 9, 2, 1, 4, 5, 3, 8, 7],
    ];

    const input = {
      unsolved: unsolved,
      solved: solved,
    };

    let dataResult = await exportCallDataPlonk(
      input,
      "./zkproof/sudoku.wasm",
      "./zkproof/sudoku_final.zkey"
    );

    // Call the function.
    let result = await sudoku.verifySudoku(
      dataResult.proof,
      dataResult.publicSignals
    );
    expect(result).to.equal(true);
  });
  it("Should be reverted on Sudoku verification because the board is not in the board list", async function () {
    this.timeout(50000);
    const unsolved = [
      [1, 2, 7, 5, 8, 4, 6, 9, 3],
      [8, 5, 6, 3, 7, 9, 1, 2, 4],
      [3, 4, 9, 6, 2, 1, 8, 7, 5],
      [4, 7, 1, 9, 5, 8, 2, 3, 6],
      [2, 6, 8, 7, 1, 3, 5, 4, 9],
      [9, 3, 5, 4, 6, 2, 7, 1, 8],
      [5, 8, 3, 2, 9, 7, 4, 6, 1],
      [7, 1, 4, 8, 3, 6, 9, 5, 2],
      [6, 9, 2, 1, 4, 5, 3, 8, 0],
    ];

    const solved = [
      [1, 2, 7, 5, 8, 4, 6, 9, 3],
      [8, 5, 6, 3, 7, 9, 1, 2, 4],
      [3, 4, 9, 6, 2, 1, 8, 7, 5],
      [4, 7, 1, 9, 5, 8, 2, 3, 6],
      [2, 6, 8, 7, 1, 3, 5, 4, 9],
      [9, 3, 5, 4, 6, 2, 7, 1, 8],
      [5, 8, 3, 2, 9, 7, 4, 6, 1],
      [7, 1, 4, 8, 3, 6, 9, 5, 2],
      [6, 9, 2, 1, 4, 5, 3, 8, 7],
    ];

    const input = {
      unsolved: unsolved,
      solved: solved,
    };

    let dataResult = await exportCallDataPlonk(
      input,
      "./zkproof/sudoku.wasm",
      "./zkproof/sudoku_final.zkey"
    );

    await expect(
      sudoku.verifySudoku(dataResult.proof, dataResult.publicSignals)
    ).to.be.reverted;
  });
});
```

When you run tests you will see something like this:

![RunTestsPlonk.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654638832142/r5dlGfX6-.png align="left")

* Change the `run.js` file inside the `scripts` folder:
    

```javascript
const main = async () => {
  const SudokuPlonkVerifier = await hre.ethers.getContractFactory(
    "SudokuPlonkVerifier"
  );
  const sudokuPlonkVerifier = await SudokuPlonkVerifier.deploy();
  await sudokuPlonkVerifier.deployed();
  console.log(
    "SudokuPlonkVerifier Contract deployed to:",
    sudokuPlonkVerifier.address
  );

  const Sudoku = await hre.ethers.getContractFactory("Sudoku");
  const sudoku = await Sudoku.deploy(sudokuPlonkVerifier.address);
  await sudoku.deployed();
  console.log("Sudoku Contract deployed to:", sudoku.address);

  let board = await sudoku.generateSudokuBoard(new Date().toString());
  console.log(board);

  let callDataSudoku = [
    "0x0cad11bcba4c82b09bad0720b4693782c443a2aee8e43b94ee7e83850dfea8b8094e422b5d68885e47f861e1ec7b60aff4c55268c45d8877afbd6b013e1a8989003b1634d339cb68c50521c250e3163c8ab870a3c232391e992fcec05ffd83650df9a701ade0b1c24c98d727c6d9ebe82bf33e89aaf7b63df9b32616b1756762151ee918cb6e3806f8506ab0b846f0e92e534ea61bf6eed851c3f28d93577448048be778315a418d7d48fe8fd5dec7697cf2a823c720451f58a12ce0fd522d3524fa7ea8fe71a6613f6ac5d7cec371fc9c29bc8e9694f7b0901dad738299cfaf1c753b100e9e614c4811be8c6495cbf6680daee844eac6569d26d30adb92975800f8cc20a55ba04162e162edea8796aab4e3918d797a3cb6960519bfbf7923be2346b0d2eae99719055b79f6732da552b3e8ec4affbfece89ca14d5640b19de825331e9ed347a6e1ce332a3c228a12e252abc465d018fb03748bf79fffd66f34208a692360905da629e9208246b99101d058d1c18fcf857ce9d1d0e686758f6b13088d4592a6915088cf7e524bf04df33a3310ac21bb856db711ef1cbc9a4fd82f58b4173d1b4312d1961bd11903b9ce53d1193f9b853cffe7b5701e6c1ddb3a30318f2ef1cb24add238e060988a7f42c5ec2719fdc72582529a1896cff1f22c073d033632137411aa5b2e50fb61f5baf865e09c55abc7502207ac3514a328ac109e4bd2ab7c9532932c685eea64ed7ca96015c898b10c9b9daa4ab50a6c0ba228501e4ac8bdf49cbaf1a3f1a3236f7c0525914f41528be1fe37c1851d02310a148df09f19bcace7d5b4893326c3b4385b50e417400cd17be1d71ac50224fd79078f558eedbc0b807179f3a68308da86f0c06c58db4e6102c76b4a6138460b690576fc6285a2514fd31673b1bfe8203693424e4523937c0c2c4830ec5b07ee302e39a8d46482f2cbb1b55073ca8f1d9daa05b6028c00461396906a400a99590725f323766c368dce2bb783d5430d4cc71fe2335eb9271e47bd99401248757570228d714d3d6ef977d3658c77a34ebb83b485368f8785ef909bd9b00172cea59d2086f4b69c8e6cf04e6111ba420f8d676d0b4273f150332b09dc28164a5429e1",
    [
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000006",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000007",
      "0x0000000000000000000000000000000000000000000000000000000000000002",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000008",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000009",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000006",
      "0x0000000000000000000000000000000000000000000000000000000000000008",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000001",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000003",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000007",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000002",
      "0x0000000000000000000000000000000000000000000000000000000000000009",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000004",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000005",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000007",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000006",
      "0x0000000000000000000000000000000000000000000000000000000000000005",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000001",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000008",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000001",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000005",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000003",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000007",
      "0x0000000000000000000000000000000000000000000000000000000000000009",
      "0x0000000000000000000000000000000000000000000000000000000000000002",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000000",
      "0x0000000000000000000000000000000000000000000000000000000000000004",
    ],
  ];

  // Call the function.
  let result = await sudokuPlonkVerifier.verifyProof(
    callDataSudoku[0],
    callDataSudoku[1]
  );

  console.log("Result", result);
};

const runMain = async () => {
  try {
    await main();
    process.exit(0);
  } catch (error) {
    console.log(error);
    process.exit(1);
  }
};

runMain();
```

* Change the `deploy.js` file inside the `scripts` folder:
    

```javascript
const main = async () => {
  const SudokuPlonkVerifier = await hre.ethers.getContractFactory(
    "SudokuPlonkVerifier"
  );
  const sudokuPlonkVerifier = await SudokuPlonkVerifier.deploy();
  await sudokuPlonkVerifier.deployed();
  console.log(
    "SudokuPlonkVerifier Contract deployed to:",
    sudokuPlonkVerifier.address
  );

  const Sudoku = await hre.ethers.getContractFactory("Sudoku");
  const sudoku = await Sudoku.deploy(sudokuPlonkVerifier.address);
  await sudoku.deployed();
  console.log("Sudoku Contract deployed to:", sudoku.address);
};

const runMain = async () => {
  try {
    await main();
    process.exit(0);
  } catch (error) {
    console.log(error);
    process.exit(1);
  }
};

runMain();
```

### Frontend changes

* In the `sudoku.js` file, inside `zksdoku-ui/pages`, change each time the `contract.verifySudoku` function is called:
    

```javascript
result = await contract.verifySudoku(calldata.proof, calldata.publicSignals);
```

* Change `snarkjsZkproof.js`, inside `zksudoku-ui/zkproof` to use Plonk:
    

```javascript
import { plonk } from "snarkjs";

export async function exportCallDataPlonk(input, wasmPath, zkeyPath) {
  const { proof: _proof, publicSignals: _publicSignals } =
    await plonk.fullProve(input, wasmPath, zkeyPath);
  const calldata = await plonk.exportSolidityCallData(_proof, _publicSignals);

  console.log("calldata", calldata);
  const calldataSplit = calldata.split(",");
  const [proof, ...rest] = calldataSplit;
  const publicSignals = JSON.parse(rest.join(",")).map((x) =>
    BigInt(x).toString()
  );
  return { proof, publicSignals };
}
```

**Note:** If you get an error importing `plonk` from `snarkjs` that says:

```text
./node_modules/fastfile/src/fastfile.js
Can't import the named export 'O_TRUNC' (imported as 'O_TRUNC') from default-exporting module (only default export is available)
```

Then, import `plonk` like so:

```javascript
const plonk = require("snarkjs").plonk;
```

Or this way:

```javascript
const { plonk } = require("snarkjs");
```

* Change the `snarkjsSudoku.js` file inside `zksudoku-ui/zkproof/sudoku` to use the `exportCallDataPlonk` function:
    

```javascript
import { exportCallDataPlonk } from "../snarkjsZkproof";

export async function sudokuCalldata(unsolved, solved) {
  const input = {
    unsolved: unsolved,
    solved: solved,
  };

  let dataResult;

  try {
    dataResult = await exportCallDataPlonk(
      input,
      "/zkproof/sudoku.wasm",
      "/zkproof/sudoku_final.zkey"
    );
  } catch (error) {
    console.log(error);
    window.alert("Wrong answer");
  }

  return dataResult;
}
```

* Using Plonk, you can see that the `sudoku_final.zkey` file is about 470 MB. You can add this file to `.gitignore` in case you do not want to commit it because of the size.
    

## Zero Knowledge Structure

The following graphic shows the structure of the most important zero knowledge elements of the zkSudoku project.

```text
├── circuits
│   ├── sudoku
│   │   ├── sudoku.circom
├── contracts
│   ├── contracts
│   │   ├── Sudoku.sol
│   │   ├── sudokuVerifier.sol
├── zksudoku-ui
│   ├── public
│   │   ├── zkproof
│   │   │   ├── sudoku.wasm
│   │   │   ├── sudoku_final.zkey
│   ├── zkproof
│   │   ├── sudoku
│   │   │   ├── snarkjsSudoku.js
│   │   ├── snarkjsZkproof.js
```

**Note:** The zero knowledge structure for Groth16 and Plonk are almost the same. The difference is the name of the solidity verifier file. In case of Groth16 is `sudokuVerifier.sol` and for Plonk is `sudokuPlonkVerifier.sol`.

## Github Repositories

* zkSudoku using Groth16: [https://github.com/vplasencia/zkSudoku](https://github.com/vplasencia/zkSudoku)
    
* zkSudoku using Plonk: [https://github.com/vplasencia/zkSudoku-plonk](https://github.com/vplasencia/zkSudoku-plonk)
    

## Live App

[https://zk-sudoku.vercel.app/](https://zk-sudoku.vercel.app/)

## Conclusions

Now we have a complete zk dapp that people can use.

We can see that using Plonk instead of Groth16 can avoid a trusted ceremony for each circuit, but Plonk is not better for the user experience because it is slower and the zkey file is quite larger.