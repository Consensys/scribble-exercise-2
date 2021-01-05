# Scribble Exercise 2

In this exercise we're going to have a look at a vulnerable ERC721 smart
contract. The contract in question (`contracts/VulnerableERC721.sol`) is
copied almost verbatim from
https://github.com/OpenZeppelin/openzeppelin-contracts, except that the
imports have been adjusted a little, a bug has been inserted, and the `_mint`
function has been made public (you'll see later why).

We'll use Scribble to annotate it with properties, and use the MythX service (and more specifically the fuzzing engine behind it) to automatically check the properties (and find the bug 🐛).

### Handy Links
Scribble Repository -> https://github.com/ConsenSys/Scribble

Mythril Repository -> https://github.com/ConsenSys/Mythril

Scribble Docs 📚 -> https://docs.scribble.codes/

MythX Dashboard -> https://dashboard.mythx.io

### Installation
```
# We'll need the mythx-cli client:
pip3 install mythx-cli

# Make sure to use node 10-12
npm install scribble --global
npm install truffle --global
npm install ganache-cli --global
```

### Setting up the target

```
git clone git@github.com:ConsenSys/scribble-exercise-2.git
cd scribble-exercise-2
```


### Finding the vulnerability
The vulnerability is reachable from the `transferFrom()` function. To find it we will add some specs to this function. A good source of specs is the docstring above `IERC721.transferFrom`:
```
    /**
     * @dev Transfers `tokenId` token from `from` to `to`.
     *
     * WARNING: Usage of this method is discouraged, use {safeTransferFrom} whenever possible.
     *
     * Requirements:
     *
     * - `from` cannot be the zero address.
     * - `to` cannot be the zero address.
     * - `tokenId` token must be owned by `from`.
     * - If the caller is not `from`, it must be approved to move this token by either {approve} or {setApprovalForAll}.
     *
     * Emits a {Transfer} event.
     */
    function transferFrom(address from, address to, uint256 tokenId) external;
```

### Adding annotations

We can extract the following specs from the above docstring:
<details>
<summary> If the transfer succeeds, then `from` is not 0.</summary>
<br>
<pre>
    /// if_succeeds {:msg "From is never 0"}  from != address(0);
    function transferFrom(address from, address to, uint256 tokenId) public virtual override {
        ...
    }
</pre>
</details>

<details>
<summary> If the transfer succeeds then `to` is not 0. (i.e. you can't destroy NFTs).</summary>
<br>
<pre>
    /// if_succeeds {:msg "Can't destroy NFTs."}  to != address(0);
    function transferFrom(address from, address to, uint256 tokenId) public virtual override {
        ...
    }
</pre>
</details>

<details>
<summary> If the transfer succeeds then `tokenId` must be owned by `from`.</summary>
<br>
<pre>
    /// if_succeeds {:msg "Correct from."}  from == old(ownerOf(tokenId));
    function transferFrom(address from, address to, uint256 tokenId) public virtual override {
        ...
    }
</pre>
</details>

<details>
<summary> If the transfer succeeds then the caller must be authorized to perform the transfer. I.e. either 1) the caller is `from` or 2) the caller is approved for this token or 3) the caller is an authorized operator for `from`.</summary>
<br>
<pre>
    /// if_succeeds {:msg "Authorized user."}  let oldOwner := old(ownerOf(tokenId)) in msg.sender == oldOwner || getApproved(tokenId) == msg.sender || isApprovedForAll(oldOwner, msg.sender);
    function transferFrom(address from, address to, uint256 tokenId) public virtual override {
        ...
    }
</pre>

Note the use of `old()` here to talk about the owner of `tokenId` **before** the transfer was executed.
</details>

### Finding the bug using fuzzing (with MythX)

To find the bug, make sure first you have added your mythx api key to your environment:

```
export MYTHX_API_KEY=<your key>
```

And then you can run:
```
mythx \
    --format simple \
    analyze contracts/VulnerableERC721.sol \
    --check-properties \
    --scribble \
    --solc-version 0.6.2
```

There's quite a bit going on in this command so lets quickly unpack it:

 * `--format simple` just specifies the output format for the issues found
 * `analyze contracts/VulnerableERC721.sol` is the actual sub-command. It instructs `mythx-cli` to send an analysis request for the specified file.
 * `--check-properties` instructs MythX to focus just on violations to user specs, and not waste time on other kind of issues (e.g. integer overflows)
 * `--scribble` tells `mythx-cli` to automatically run `scribble` on the target contracts to instrument them, before submitting to the API
 * `--solc-version 0.6.2` - mythx-cli will complain if you dont' explicitly give it the Solidity version

After a short while, you should see some issues prop up. Which property was violated? Now knowing which property was violated, can you see where in the code the bug was introduced :)?