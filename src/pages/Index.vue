<template>
  <div>
    <q-list bordered class="flat q-mx-lg q-mt-md">
      <q-expansion-item
        expand-separator
        label="Proximos jogos:"
        :caption="`Jogos encontrados em ${matchesFound.length} campeonatos diferentes`"
      >
        <q-card class="q-pa-none">
          <q-card-section
            v-for="(match, idx) in matchesFound"
            :key="idx"
            class="q-pa-xs"
          >
            <q-banner rounded class="bg-grey-3">
              <template v-slot:avatar>
                <img :src="match.competitionFlag" alt="banner da competição" class="banner-image"/>
              </template>
              <div class="text-h6">{{ match.competitionName }}</div>
              <ul>
                <li v-for="(match, idx) in match.matches" :key="idx">
                  {{ match.homeTeam }} vs {{ match.awayTeam }} -
                  {{ formatDate(match.matchData) }}
                </li>
              </ul>
            </q-banner>
          </q-card-section>
        </q-card>
      </q-expansion-item>
    </q-list>
    <q-table
      title="Partidas de times da casa com potencial grande de mais de 1.5 gols:"
      :data="partidasBoas"
      :columns="columns"
      row-key="name"
      class="q-mx-lg q-my-lg"
      :loading="loading"
    >
    <template v-slot:loading>
      <q-inner-loading showing color="primary" label="Buscando os melhores resultados..." />
    </template>
  </q-table>
  </div>
</template>

<script>
import axios from "axios";

export default {
  data() {
    return {
      upcomingMatches: [],
      apiKey: "bfd74489962d4c70ad488cef72e4f000",
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
          label: "Time",
          align: "center",
          field: (row) => row.casa,
          format: (val) => `${val}`,
          sortable: true,
        },
        {
          name: "count",
          label: "Total de gols nos últimos 5 jogos",
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
          format: (val) => `${val / 5}`,
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
      loading: false
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

        console.log("this.upcomingMatches", this.upcomingMatches);

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
        this.loading = true

        for (const upcomingMatch of this.upcomingMatches) {
          const matchEndPoint = `https://api.football-data.org/v2/teams/${upcomingMatch.homeTeam.id}/matches?status=FINISHED&limit=5`;

          await this.getLastResultsWithDelay(matchEndPoint, upcomingMatch);
          await delay(5000); // Aguarde 5 segundos entre cada solicitação
        }

        this.loading = false
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

        if (!response) {
          throw new Error(
            this.$q.notify({
              message: `Erro na solicitação 2: ${response.status} - ${response.statusText}`,
              color: "red",
            })
          );
        }

        const partidas = response.data.matches;

        let count = 0;
        let casa = "";
        let fora = "";
        let qtdGolsCasa = "";
        let qtdGolsFora = "";
        let competition = "";

        partidas.forEach((partida) => {
          competition = partida.competition.name;

          if (partida.homeTeam.id === currentMatch.homeTeam.id) {
            casa = `${partida.homeTeam.name} vs ${partida.awayTeam.name}`;
            fora = partida.awayTeam.name;
            qtdGolsCasa = partida.score.fullTime.homeTeam;
            qtdGolsFora = partida.score.fullTime.awayTeam;
          } else {
            casa = `${partida.homeTeam.name} vs ${partida.awayTeam.name} 1`;
            fora = partida.homeTeam.name;
            qtdGolsCasa = partida.score.fullTime.awayTeam;
            qtdGolsFora = partida.score.fullTime.homeTeam;
          }

          count += qtdGolsCasa;
        });

        const target = 1.9;

        if (count / 5 > target) {
          const data = {
            competition,
            casa,
            count,
            dataPartida: currentMatch.utcDate
          }

          this.partidasBoas.push(data);

          console.log(
            "-----------------------INÍCIO---------------------------"
          );
          console.log(`${competition}: ${casa} ${fora}`);
          console.log("Total de gols nos últimos 5 jogos:", count);
          console.log("Média de gols por partida:", count / 5);
          console.log("-----------------------FIM---------------------------");
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
