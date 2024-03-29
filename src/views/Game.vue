<template>
    <div>
        <b-container class="container">
            <b-row>
                <b-col>{{gameMode}}</b-col>
                <b-col>Amount of Sets <b-badge pill variant="dark">{{setCounter}}</b-badge></b-col>
                <b-col>Points <b-badge pill variant="dark">{{points}}</b-badge></b-col>
            </b-row>
            <b-row>
                <b-col>
                    <b-progress height="11px" :max=5 :value="clickedCardCounter" variant="success" key="success"/>
                </b-col>
            </b-row>
            <b-row>
                <b-col>
                    <b-button v-if="noMoreSets" size="sm" class="mt-3" variant="warning" block @click="resetBoard">shuffle board
                    </b-button>
                </b-col>
            </b-row>
        </b-container>
        <canvas
                @click="clickOnCanvas($event)"
                :width="canvas_width"
                :height="canvas_height"
                id="canvas"
                ref="game_canvas"
        />
        <b-modal ref="highscore-modal" hide-footer title="Congratulation you did it!">
            <form ref="form" @submit.stop.prevent="saveScore">
                <b-form-group
                        label="Please enter your name"
                        label-for="name-input"
                        invalid-feedback="Name is required"
                >
                <b-form-input
                        id="name-input"
                        v-model="name"
                        required
                />
                </b-form-group>
            </form>
            <b-button class="mt-3" variant="success" block @click="saveScore">save</b-button>
        </b-modal>
    </div>
</template>

<script>
    import { Card } from '../assets/card/card.js';
    import { Score } from '../assets/score/score.js';
    import { Settings } from '../assets/settings/settings.js';
    import { Points } from '../assets/points/points.js';
    import { Game } from '../assets/game/game.js';

    const points = new Points();
    const score = new Score();
    const settings = new Settings();
    const game = new Game();
    const LOCAL_STORAGE_DATA_SETTINGS = 'settings';

    export default {
        name: "Game",
        data() {
            return {
                hardMode: settings.getHardmodeFlag(LOCAL_STORAGE_DATA_SETTINGS),
                gameMode: settings.getGameMode(LOCAL_STORAGE_DATA_SETTINGS), 
                name: settings.getUserName(LOCAL_STORAGE_DATA_SETTINGS),
                setCounter: 0,
                clickedCards: [],
                canvas_height: window.innerHeight / 2,
                canvas_width: window.innerWidth,
                points: 0,
                pointsForCurrentSet: points.forCurrentSetWithModes(),
                removePointsCounter: points.removeWithModes(),
                noMoreSets: false,
                clickedCardCounter: 5, // Full Processbar at the beginning
                intervalIdForClickedCard: '',
                intervalIdForCurrentSetCounter: ''
            };
        },
        methods: {
            saveScore(){
                score.save(this.name, this.points, this.gameMode);
                this.$router.push('highscore');
            },
            showModal() {
                this.$refs['highscore-modal'].show();
            },
            // TODO: Add a save Button for the current Game
            findAllSets() {
                this.setCounter = game.calculateNumberOfSets();

                // When there is no Set left show a Button to reset the board
                if (this.setCounter === 0) {
                    this.noMoreSets = true;
                }
                // When the Game is over open the Modal for the play name
                if (this.setCounter === 0 && game.allCards.length === 0) {
                    this.showModal();
                }
            },
            resetBoard() {
                for (const boardCard of game.board) {
                    game.allCards.push(boardCard);
                }
                this.createNewBoard();
                this.noMoreSets = false;
            },
            createNewBoard() {
                game.board = [];
                for (let i = 0; i < 12; i++) {
                    game.board.push(this.getRandomCard());
                }
                this.drawBoard(); // Draws the Cards on the board
                this.findAllSets(); // Finds all Sets on Board
            },
            clickOnCanvas(event) {
                const rect = this.canvas.getBoundingClientRect();
                game.x = event.clientX - rect.left;
                game.y = event.clientY - rect.top;
                this.addAndRedrawSelectedCard();

                // Starts an Interval when an Card is clicked. So the User has only 5 Seconds to click the other cards
                if (this.clickedCardCounter === 5) {
                    this.clickedCardInterval();
                    this.clickedCardCounter -= 0.1;
                }

                if (this.clickedCards.length === 3) {
                    clearInterval(this.intervalIdForClickedCard);
                    this.clickedCardCounter = 5; // Resets the clickedCard Interval
                    // When a Set is found replace the old Cards with new Cards and redraw the board
                    if (game.checkThreeCardsForASet(this.clickedCards[0], this.clickedCards[1], this.clickedCards[2])) {
                        this.points += this.pointsForCurrentSet; // Adds points for the correct Set
                        if (game.allCards.length >= 3) {
                            for (const clickedCard of this.clickedCards) {
                                game.board.splice(clickedCard, 1, this.getRandomCard()); // Set the new Card on the Board
                            }
                        } else {
                            // When the last sets are taken from the Board and there are no cards left in the stock
                            for (const clickedCard of this.clickedCards) {
                                let img = new Image(); // create new Image
                                img.src = require("@/assets/cards_svg/3333.svg");
                                const emptyCard = new Card(3, 3, 3, 3, img); // Create empty Card
                                game.board.splice(clickedCard, 1, emptyCard); // Set the new Card on the Board
                            }
                        }
                        this.findAllSets();
                        this.pointsForCurrentSet = points.forCurrentSetWithModes(); // Reset the Points for one Set
                    } else {
                        this.removePointsForCurrentSet();
                    }
                    // Reset Clicked Cards when the selected cards are not a set
                    this.resetClickedCards();
                }
            },
            resetClickedCards() {
                for (const clickedCard of this.clickedCards) {
                    this.redrawCardAfterSelcted(clickedCard, "#d9d9d9");
                }
                this.clickedCards = []; // Reset the clicked Cards
            },
            getRandomCard() {
                const number = Math.floor(Math.random() * 100) % game.allCards.length;
                const card = game.allCards[number];
                game.allCards.splice(game.allCards.indexOf(card), 1);
                return card;
            },
            drawBoard: function () {
                this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);

                for (let i = 0; i < 12; i++) {
                    const card = this.getCardPosition(game.board[i], i);
                    this.drawCard(card, card.x_max, card.y_max, "#d9d9d9");
                }
            },
            drawCard: function (card, dWidth, dHeight, color) {
                // Putting the image and its coordinates on the canvas
                const self = this;
                const imagesrc = card.image.src;

                card.image.src = imagesrc // load the image
                card.image.onload = function () {
                    // Clear the blue box Top and Bottom
                    self.ctx.fillStyle = color;
                    self.ctx.clearRect(card.x_min + 1, card.y_min + 1, dWidth, dHeight);
                    self.ctx.clearRect(card.x_min - 1, card.y_min - 1, dWidth, dHeight);
                    self.ctx.fillRect(card.x_min, card.y_min, dWidth, dHeight);
                    self.ctx.drawImage(card.image, card.x_min, card.y_min, dWidth, dHeight);
                    card.setPosition(card.x_min, card.y_min, (dWidth + card.x_min), (dHeight + card.y_min));
                }
            },
            removePointsForCurrentSet() {
                if (this.pointsForCurrentSet >= 5) {
                    this.pointsForCurrentSet -= 5;
                }
            },
            getCardPosition(card, postionOnBoard) {
                const height = this.canvas.height / 4;
                const width = this.canvas.width / 3;
                switch (postionOnBoard) {
                    case 0:
                        card.setPosition(0, 0, width, height);
                        break;
                    case 1:
                        card.setPosition(width, 0, width, height);
                        break;
                    case 2:
                        card.setPosition(width * 2, 0, width, height);
                        break;
                    case 3:
                        card.setPosition(0, height, width, height);
                        break;
                    case 4:
                        card.setPosition(width, height, width, height);
                        break;
                    case 5:
                        card.setPosition(width * 2, height, width, height);
                        break;
                    case 6:
                        card.setPosition(0, height * 2, width, height);
                        break;
                    case 7:
                        card.setPosition(width, height * 2, width, height);
                        break;
                    case 8:
                        card.setPosition(width * 2, height * 2, width, height);
                        break;
                    case 9:
                        card.setPosition(0, height * 3, width, height);
                        break;
                    case 10:
                        card.setPosition(width, height * 3, width, height);
                        break;
                    case 11:
                        card.setPosition(width * 2, height * 3, width, height);
                        break;
                }
                return card;
            },
            redrawCardAfterSelcted(cardIndex, color) {
                let card = game.board[cardIndex];
                card = this.getCardPosition(card, cardIndex); // Set Position
                card.setPosition(card.x_min, card.y_min, card.x_max + card.x_min, card.y_max + card.y_min);
                this.drawCard(card,card.x_max - card.x_min, card.y_max - card.y_min, color);
            },
            addAndRedrawSelectedCard() {
                const cardIndex = game.getClickedCard();
                if (!this.clickedCards.includes(cardIndex)) {
                    this.clickedCards.push(cardIndex); // Add Card to the "checkForSet" Array
                    this.redrawCardAfterSelcted(cardIndex, "blue");
                }
            },
            startSetInterval() {
                const self = this;
                this.intervalIdForCurrentSetCounter = setInterval(function () {
                    if (self.pointsForCurrentSet - self.removePointsCounter >= 0) {
                        self.pointsForCurrentSet -= self.removePointsCounter;
                    }
                }, 1000);
            },
            clickedCardInterval() {
                const self = this;
                this.intervalIdForClickedCard = setInterval(function () {
                    if (self.clickedCardCounter - 0.1 >= 0) {
                        self.clickedCardCounter -= 0.1;
                    } else {
                        self.clickedCardCounter = 5;
                        clearInterval(self.intervalIdForClickedCard);
                        self.removePointsForCurrentSet();
                        self.resetClickedCards();
                    }
                }, 100);
            },
        },
        beforeDestroy () {
            // Stops Timer for current Set 
            clearInterval(this.intervalIdForCurrentSetCounter);
        },
        mounted: function () {
            game.createAllCards(); // Also Draws the First Board and Find all possible Sets

            this.createNewBoard(); // Creates random board of cards

            this.startSetInterval();
            const self = this;
            window.addEventListener("resize", function () {
                self.canvas.width = window.innerWidth;
                self.canvas.height = window.innerHeight / 2;
                self.drawBoard();
            });
        },
        computed: {
            canvas: function () {
                return this.$refs.game_canvas;
            },
            ctx: function () {
                return this.canvas.getContext('2d');
            }
        }
    };
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
    canvas {
        background-color: #d9d9d9;
        display: block;
        position: absolute;
        overflow: hidden;
        bottom: 0;
        width: 100%;
        height: 50%;
    }
    .container {
        width: 100%;
        height: 50%;
    }
</style>