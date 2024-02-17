<template>
  <div>
    <q-list bordered class="flat q-mx-lg q-mt-md">
      <q-expansion-item
        class="text-center text-h6"
        icon="sports_soccer"
        :label="`Foram encontradas ${upcomingMatches.length} partidas em ${matchesFound.length} campeonatos diferentes`"
        header-class="bg-teal text-white"
        expand-icon-class="text-white"
        expand-separator
      >
        <q-card class="q-pa-none bg-teal-2">
          <q-card-section
            v-for="(match, idx) in matchesFound"
            :key="idx"
            class="q-pa-xs"
          >
            <q-banner rounded class="bg-teal-1">
              <template v-slot:avatar>
                <img
                  :src="match.competitionFlag"
                  alt="banner da competição"
                  class="banner-image"
                />
              </template>
              <div class="text-h6">{{ match.competitionName }}</div>
              <div v-for="(match, idx) in match.matches" :key="idx">
                {{ match.homeTeam }} vs {{ match.awayTeam }} -
                {{ formatDate(match.matchData) }}
              </div>
            </q-banner>
          </q-card-section>
        </q-card>
      </q-expansion-item>
    </q-list>

    <q-separetor></q-separetor>

    <div class="q-px-lg q-pt-lg q-pb-sm text-h5 text-center" v-if="loading">
      Buscando os melhores resultados... {{ verifiedMatches }}/{{
        upcomingMatches.length + 1
      }}
    </div>
    <div v-if="loading" class="q-px-lg">
      <q-markup-table>
        <thead>
          <tr>
            <th class="text-left" style="width: 120px">
              <q-skeleton animation="blink" type="text" />
            </th>
            <th class="text-right">
              <q-skeleton animation="blink" type="text" />
            </th>
            <th class="text-right">
              <q-skeleton animation="blink" type="text" />
            </th>
            <th class="text-right">
              <q-skeleton animation="blink" type="text" />
            </th>
            <th class="text-right">
              <q-skeleton animation="blink" type="text" />
            </th>
            <th class="text-right">
              <q-skeleton animation="blink" type="text" />
            </th>
          </tr>
        </thead>

        <tbody>
          <tr v-for="n in 5" :key="n">
            <td class="text-left">
              <q-skeleton animation="blink" type="text" width="85px" />
            </td>
            <td class="text-right">
              <q-skeleton animation="blink" type="text" width="50px" />
            </td>
            <td class="text-right">
              <q-skeleton animation="blink" type="text" width="35px" />
            </td>
            <td class="text-right">
              <q-skeleton animation="blink" type="text" width="65px" />
            </td>
            <td class="text-right">
              <q-skeleton animation="blink" type="text" width="25px" />
            </td>
            <td class="text-right">
              <q-skeleton animation="blink" type="text" width="85px" />
            </td>
          </tr>
        </tbody>
      </q-markup-table>
    </div>
    <div v-else>
      <div class="q-px-lg q-pt-lg q-pb-sm text-h5 text-center">
        Partidas de times da casa com potencial para mais de 1,5 gols
      </div>
      <q-table
        :data="partidasBoas"
        :columns="columns"
        row-key="name"
        class="q-mx-lg"
        :loading="loading"
        dark
      >
        <template v-slot:loading>
          <q-inner-loading
            showing
            color="primary"
            :label="`Buscando os melhores resultados... ${verifiedMatches}/${upcomingMatches.length}`"
          />
        </template>
      </q-table>
    </div>
  </div>
</template>

<script>
import axios from "axios";

export default {
  data() {
    return {
      upcomingMatches: [],
      apiKey: "bfd74489962d4c70ad488cef72e4f000",
      generalTimeOut: 6000,
      numberVerifiedMatches: 5,
      matchesFound: [],
      partidasBoas: [],
      columns: [
        {
          name: "competition",
          label: "Campeonato",
          align: "center",
          field: (row) => row.competition,
          format: (val) => `${val}`,
          sortable: true,
        },
        {
          name: "casa",
          label: "Times",
          align: "center",
          field: (row) => row.casa,
          format: (val) => `${val}`,
          sortable: true,
        },
        {
          name: "count",
          label: `Total de gols nos últimos 5 jogos`,
          align: "center",
          field: (row) => row.count,
          format: (val) => `${val}`,
          sortable: true,
        },
        {
          name: "count",
          label: "Média de gols por partida",
          align: "center",
          field: (row) => row.count,
          format: (val) => `${val / this.numberVerifiedMatches}`,
          sortable: true,
        },
        {
          name: "halfTimeGols",
          label: "Quantidade de partidas com gol no 1º tempo",
          align: "center",
          field: (row) => row.halfTimeGols,
          format: (val) => `${val}/5`,
          sortable: true,
        },
        {
          name: "dataPartida",
          label: "Data da partida",
          align: "center",
          field: (row) => row.dataPartida,
          format: (val) => this.formatDate(val),
          sortable: true,
        },
      ],
      loading: false,
      verifiedMatches: 0,
    };
  },
  created() {
    this.formatCurrentDays();
  },
  methods: {
    formatCurrentDays() {
      const today = new Date();
      const tomorrow = new Date(today);
      tomorrow.setDate(tomorrow.getDate() + 1);
      const todayDateString = today.toISOString().split("T")[0];
      const tomorrowDateString = tomorrow.toISOString().split("T")[0];

      this.getUpcomingMatches(todayDateString, tomorrowDateString);
    },
    async getUpcomingMatches(todayDateString, tomorrowDateString) {
      try {
        const endpoint = `https://api.football-data.org/v2/matches?dateFrom=${todayDateString}&dateTo=${tomorrowDateString}`;

        const response = await axios.get(endpoint, {
          headers: {
            "X-Auth-Token": this.apiKey,
          },
        });

        /** Filtra apenas partidas com status 'Agendada' */
        this.upcomingMatches = response.data.matches.filter(
          (match) => match.status === "SCHEDULED"
        );

        this.mountHeaderCurrentMatches();
      } catch (error) {
        this.showErrorNotification(error);
      }
    },
    mountHeaderCurrentMatches() {
      this.upcomingMatches.forEach((x) => {
        // Verifica se this.matchesFound está vazio
        if (this.matchesFound.length === 0) {
          // Se estiver vazio, cria um novo objeto de dados e adiciona à this.matchesFound
          const matchInfo = {
            competitionName: x.competition.name,
            competitionFlag: x.competition.area.ensignUrl,
            matches: [
              {
                awayTeam: x.awayTeam.name,
                homeTeam: x.homeTeam.name,
                matchData: x.utcDate,
              },
            ],
          };
          this.matchesFound.push(matchInfo);
        } else {
          // Se não estiver vazio, verifica se já existe um objeto com o mesmo nome de competição
          let found = false;
          this.matchesFound.forEach((matchInfo) => {
            if (matchInfo.competitionName === x.competition.name) {
              // Se já existir um objeto com o mesmo nome de competição, adiciona a partida ao array de partidas
              matchInfo.matches.push({
                awayTeam: x.awayTeam.name,
                homeTeam: x.homeTeam.name,
                matchData: x.utcDate,
              });
              found = true;
            }
          });
          // Se não encontrou um objeto com o mesmo nome de competição, adiciona um novo objeto de dados
          if (!found) {
            const matchInfo = {
              competitionName: x.competition.name,
              competitionFlag: x.competition.area.ensignUrl,
              matches: [
                {
                  awayTeam: x.awayTeam.name,
                  homeTeam: x.homeTeam.name,
                  matchData: x.utcDate,
                },
              ],
            };
            this.matchesFound.push(matchInfo);
          }
        }
      });

      this.searchLatestScoresHomeTime();
    },
    async searchLatestScoresHomeTime() {
      try {
        const delay = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
        this.loading = true;

        for (const upcomingMatch of this.upcomingMatches) {
          const matchEndPoint = `https://api.football-data.org/v2/teams/${upcomingMatch.homeTeam.id}/matches?status=FINISHED&limit=${this.numberVerifiedMatches}`;

          this.verifiedMatches++;

          await this.getLastResultsWithDelay(matchEndPoint, upcomingMatch);
          await delay(this.generalTimeOut); // Aguarde 5 segundos entre cada solicitação
        }

        this.loading = false;
      } catch (error) {
        this.showErrorNotification(error);
      }
    },
    async getLastResultsWithDelay(matchEndPoint, currentMatch) {
      try {
        const response = await axios.get(matchEndPoint, {
          headers: {
            "X-Auth-Token": this.apiKey,
          },
        });

        const matches = response.data.matches;

        let count = 0;
        let casa = "";
        let qtdGolsCasa = "";
        let competition = "";
        let halfTimeGols = 0;

        matches.forEach((match) => {
          const halfTimeResult = match.score;
          competition = match.competition.name;

          if (match.homeTeam.id === currentMatch.homeTeam.id) {
            casa = `${currentMatch.awayTeam.name} vs ${currentMatch.homeTeam.name}`;
            qtdGolsCasa = match.score.fullTime.homeTeam;
          } else {
            casa = `${currentMatch.homeTeam.name} vs ${currentMatch.awayTeam.name}`;
            qtdGolsCasa = match.score.fullTime.awayTeam;
          }

          if (
            halfTimeResult.halfTime.awayTeam > 0 ||
            halfTimeResult.halfTime.homeTeam > 0
          ) {
            halfTimeGols++;
          }

          count += qtdGolsCasa;
        });

        const target = 2;

        if (count / this.numberVerifiedMatches > target) {
          const data = {
            competition,
            casa,
            count,
            dataPartida: currentMatch.utcDate,
            halfTimeGols,
          };

          this.partidasBoas.push(data);
        }
      } catch (error) {
        this.showErrorNotification(error);
      }
    },
    formatDate(dataString) {
      const data = new Date(dataString);
      const day = data.getDate().toString().padStart(2, "0");
      const month = (data.getMonth() + 1).toString().padStart(2, "0");
      const year = data.getFullYear().toString();
      return `${day}/${month}/${year}`;
    },
    showErrorNotification(error) {
      this.$q.notify({
        message: `Erro na solicitação: ${error.message}`,
        color: "red",
        position: "top",
      });
    },
  },
};
</script>
<style lang="sass" scoped>
.banner-image
  width: 100px
  height: 64px
</style>
