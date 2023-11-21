# poll-html

``index.html``
```
<!DOCTYPE html>
<html lang="en">
	<head>
		<meta charset="UTF-8" />
		<meta name="viewport" content="width=device-width, initial-scale=1.0" />
		<title>Simple poll dapp hosted on an ICP canister smart contract</title>

		<style>
			body {
				font-family: Arial, sans-serif;
				margin: 0;
				padding: 0;
			}

			.container {
				max-width: 800px;
				margin: 0 auto;
				padding: 20px;
				position: relative;
			}

			.title-container {
				border: 2px solid #007bff;
				background-color: #f0f0f0;
				padding: 20px;
				border-radius: 5px;
			}

			h1 {
				font-size: 32px;
				margin-bottom: 20px;
				text-align: center;
				margin-top: 0;
			}

			h2 {
				font-size: 24px;
				margin-bottom: 10px;
				text-align: center;
			}

			form {
				margin-bottom: 20px;
				border: 2px solid #8bc34a;
				padding: 20px;
				border-radius: 5px;
			}

			label {
				display: block;
				margin-bottom: 10px;
				font-size: 18px;
				text-align: left;
			}

			input[type="radio"] {
				margin-right: 5px;
			}

			button {
				padding: 10px 20px;
				background-color: #007bff;
				border: none;
				color: #fff;
				font-size: 18px;
				cursor: pointer;
				border-radius: 5px;
			}

			button#reset {
				background-color: #dc3545;
				position: absolute;
				bottom: 20px;
				left: 20px;
				margin: 10px 0; /* Add margin */
			}

			button:hover {
				background-color: #0056b3;
			}

			#results {
				margin-top: 20px;
				font-size: 18px;
				border: 2px solid #8bc34a;
				padding: 20px;
				border-radius: 5px;
				position: relative;
			}
		</style>
	</head>
	<body>
		<div class="container">
			<div class="title-container">
				<h1>Simple Voting Poll</h1>
			</div>
			<h2 id="question">Sample Question</h2>

			<!-- Form where users vote -->
			<div class="form-container">
				<form id="radioForm">
					<label>
						<input type="radio" name="option" value="Rust" />
						Rust </label
					><br />
					<label>
						<input type="radio" name="option" value="Motoko" />
						Motoko </label
					><br />
					<label>
						<input type="radio" name="option" value="TypeScript" />
						TypeScript </label
					><br />
					<label>
						<input type="radio" name="option" value="Python" />
						Python </label
					><br />
					<button type="submit">Vote</button>
				</form>
			</div>

			<!-- Poll results appear here-->
			<h2 id="results-title">Results</h2>
			<div id="results"></div>
		</div>
		<button id="reset">Reset Poll</button>
	</body>
</html>
```

``index.js``

```
const pollForm = document.getElementById("radioForm");
const resultsDiv = document.getElementById("results");
const resetButton = document.getElementById("reset");

import { poll_backend } from "../../declarations/poll_backend";

const pollResults = {
	Rust: 0,
	Motoko: 0,
	Typescript: 0,
	Python: 0,
};

document.addEventListener(
	"DOMContentLoaded",
	async (e) => {
		e.preventDefault();

		const question = await poll_backend.getQuestion();
		document.getElementById("question").innerText = question;

		const voteCounts = await poll_backend.getVotes();
		updateLocalVoteCounts(voteCounts);
		displayResults();
		return false;
	},
	false
);

pollForm.addEventListener(
	"submit",
	async (e) => {
		e.preventDefault();

		const formData = new FormData(pollForm);
		const checkedValue = formData.get("option");

		const updatedVoteCounts = await poll_backend.vote(checkedValue);
		console.log("Returning from await...");
		console.log(updatedVoteCounts);
		updateLocalVoteCounts(updatedVoteCounts);
		displayResults();
		return false;
	},
	false
);

resetButton.addEventListener(
	"click",
	async (e) => {
		e.preventDefault();

		await poll_backend.resetVotes();
		const voteCounts = await poll_backend.getVotes();
		updateLocalVoteCounts(voteCounts);

		displayResults();
		return false;
	},
	false
);

function displayResults() {
	let resultHTML = "<ul>";
	for (let key in pollResults) {
		resultHTML +=
			"<li><strong>" + key + "</strong>: " + pollResults[key] + "</li>";
	}
	resultHTML += "</ul>";
	resultsDiv.innerHTML = resultHTML;
}

function updateLocalVoteCounts(arrayOfVoteArrays) {
	for (let voteArray of arrayOfVoteArrays) {
		//Example voteArray -> ["Motoko","0"]
		let voteOption = voteArray[0];
		let voteCount = voteArray[1];
		pollResults[voteOption] = voteCount;
	}
}```

``main.mo``

```
import Text "mo:base/Text";
import RBTree "mo:base/RBTree";
import Iter "mo:base/Iter";
import Nat "mo:base/Nat";

actor {
  var question: Text = "What is your favorite programming Language?";
  var votes: RBTree.RBTree<Text, Nat> = RBTree.RBTree(Text.compare);
  

  public query func getQuestion() : async Text {
    question
  };

  public query func getVotes() : async [(Text, Nat)] {
    // Iter -> pointer like DS allowing for DS values to be parsed one by one in a sequential manner.
    // Iter.toArray() -> converts Iter<(Text,Nat)> to [(Text,Nat)];
    Iter.toArray(votes.entries());
  };

  public func vote(entry: Text): async [(Text,Nat)] {
    // Check if the entry already has votes.
    // Note that "votes_for_entry" is of type ?Nat. This is because:
    // * If the entry is in the RBTree, the RBTree returns a number.
    // * If the entry is not in the RBTree, the RBTree returns `null` for the new entry.
    let votes_for_entry :?Nat = votes.get(entry);

    let current_votes_for_entry: Nat = switch votes_for_entry {
      case null 0;
      case (?Nat) Nat;
    };

    votes.put(entry, current_votes_for_entry + 1);

    Iter.toArray(votes.entries());
  };

  public func resetVotes() : async [(Text,Nat)] {
    votes.put("Motoko", 0);
    votes.put("Rust", 0);
    votes.put("Typescript", 0);
    votes.put("Python", 0);
    Iter.toArray(votes.entries());
  }
}```
