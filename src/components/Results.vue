<template>
    <h2 v-if="races.ResultList">{{  races.ResultList[0].Name }}</h2>
    <h2 v-if="registrations.RaceInfo">{{  registrations.RaceInfo.Name }}</h2>
    <v-container fluid>
        <v-row row wrap>
            <v-col v-for="group in results.Results" cols="2">
                <v-card>
                    <v-card-item>
                        <v-card-title>{{ getTitle(group) }}</v-card-title>
                    </v-card-item>
                    <v-card-text>
                        <pre>
{{ getText(group) }}
                        </pre>
                    </v-card-text>
                </v-card>
            </v-col>
        </v-row>
        <v-row v-if="results.Results" row wrap>
            <v-col cols="12">
                <v-card>
                    <v-card-text>
                        {{ getRaffle() }}
                    </v-card-text>
                </v-card>
            </v-col>
        </v-row>
    </v-container>
</template>


<script>
export default {
    data() {
        return {
            results: {},
            races: {},
            timer: {},
            registrations: {},
            winners: new Set()
        }
    },
    methods: {
        getRaces() {
            fetch('https://freedomrun.bluetor.ch/race')
                .then(response => response.json())
                .then(data => this.races = data);
            fetch('https://freedomrun.bluetor.ch/list')
                .then(response => response.json())
                .then(data => this.registrations = data);
            setTimeout(this.getResults, 5000);
            this.timer = setInterval(this.getResults, 60000);
        },
        getResults() {
            console.log("results");
            fetch('https://freedomrun.bluetor.ch/race/' + this.races.ResultList[0].RaceId)
                .then(response => response.json())
                .then(data => this.results = data);
        },
        getTitle(group) {
            let title = "";
            if (group.Grouping.Overall) {
                title = "OVERALL";
            } else {
                title = group.Grouping.Category
            }
            if (group.Grouping.Gender) {
                title+= " / " + group.Grouping.Gender;
            }
            return title;
        },
        getText(group) {
            let text = ""
            for (let i = 0; i < Math.min(3, group.Racers.length); i++) {
                let racer = group.Racers[i];
                if (racer.ChipTime !== "DNS" && racer.ChipTime !== "DNF") {
                    text+= racer.ChipTime + " " + racer.Bib + " " + racer.Name + "\n";
                    this.winners.add(racer.Bib);
                }
            }
            return text;
        },
        getRaffle() {
            console.log("raffle");
            let raffle = new Set();
            if (localStorage.raffle) {
                let raffleArray = JSON.parse(localStorage.raffle);
                raffle = new Set(raffleArray);
                let overlap = new Set();
                for (const runner of raffle) {
                    if (this.winners.has(runner.Bib.toString())) {
                        overlap.add(runner);
                    }
                }
                for (const runner of overlap) {
                    raffle.delete(runner);
                }
            } 
            if (raffle.size < 75) {
                console.log("raffle addition");
                if (this.registrations.StartList) {
                    console.log("start list ready");
                    let eligible = this.registrations.StartList.filter(runner => runner.Distance === "5K");
                    let currentIndex = eligible.length, randomIndex;
                    while (currentIndex != 0) {
                        randomIndex = Math.floor(Math.random() * currentIndex);
                        currentIndex--;
                        [eligible[currentIndex], eligible[randomIndex]] = [
                        eligible[randomIndex], eligible[currentIndex]];
                    }
                    let i = 0;
                    while (raffle.size < 75 && i < eligible.length) {
                        if (!this.winners.has(eligible[i].Bib.toString()) && !raffle.has(eligible[i])) {
                            raffle.add(eligible[i]);
                        }
                        i++;
                    }
                    localStorage.raffle = JSON.stringify([...raffle]);
                }
            }
            let raffleText = "";
            const primaryUnsorted = Array.from(raffle).slice(0,49);
            const primarySorted = primaryUnsorted.sort((a, b) => a.Bib - b.Bib);
            for (const runner of primarySorted) {
                raffleText+= "/ " + runner.Bib + "-" + runner.Name + " ";
            }
            const secondaryUnsorted = Array.from(raffle).slice(50);
            const secondarySorted = secondaryUnsorted.sort((a, b) => a.Bib - b.Bib);
            raffleText+= "/ *** ALTERNATES *** ";
            for (const runner of secondarySorted) {
                raffleText+= "/ " + runner.Bib + "-" + runner.Name + " ";
            }
            return raffleText;
        }
    },
    mounted() {
        this.getRaces();
    },
    beforeUnmount() {
        clearInterval(this.timer);
        console.log("timer cleared");
    }
}
</script>