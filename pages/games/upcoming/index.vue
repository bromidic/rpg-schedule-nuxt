<template>
  <v-container fluid>
    <v-app-bar dense class="mb-3 hidden-sm-and-up">
      <v-text-field
        v-model="searchQuery"
        @keyup="search"
        flat
        solo
        prepend-inner-icon="mdi-magnify"
        style="height: 48px; margin-left: -16px;"
      ></v-text-field>
      <v-btn
        text
        small
        v-if="guilds.filter(g => g.collapsed).length == guilds.length"
        @click="expandAll"
        class="ml-4"
      >
        <v-icon>mdi-chevron-double-up</v-icon>
      </v-btn>
      <v-btn
        text
        small
        v-if="guilds.filter(g => !g.collapsed).length > 0"
        @click="collapseAll"
        class="ml-4"
      >
        <v-icon>mdi-chevron-double-down</v-icon>
      </v-btn>
    </v-app-bar>
    <v-app-bar dense class="mb-3 hidden-xs-only">
      <v-text-field
        v-model="searchQuery"
        @keyup="search"
        flat
        solo
        prepend-inner-icon="mdi-magnify"
        style="height: 48px; margin-left: -16px;"
      ></v-text-field>
      <v-btn
        text
        small
        v-if="guilds.filter(g => g.collapsed).length > 0"
        @click="expandAll"
        class="ml-4"
      >Expand All</v-btn>
      <v-btn
        text
        small
        v-if="guilds.filter(g => !g.collapsed).length > 0"
        @click="collapseAll"
        class="ml-4"
      >Collapse All</v-btn>
    </v-app-bar>

    <v-card
      v-for="(guild, g) in guilds.filter(g => !g.filtered && !!g.games.find(game => !game.deleted)).filter(g => g.games.filter(gm => !gm.filtered).length > 0)"
      v-bind:key="g"
      max-width="100%"
      class="mb-3"
    >
      <v-toolbar dense color="discord">
        <v-img
          v-if="guild.icon"
          :src="guild.icon"
          max-width="30"
          class="mr-3 hidden-xs-only"
          style="border-radius: 50%;"
        ></v-img>
        <v-toolbar-title>{{guild.name}}</v-toolbar-title>

        <v-spacer></v-spacer>

        <v-btn icon @click="guild.collapsed = !guild.collapsed">
          <v-icon v-if="!guild.collapsed">mdi-chevron-down</v-icon>
          <v-icon v-if="guild.collapsed">mdi-chevron-up</v-icon>
        </v-btn>

        <v-menu left offset-y>
          <template v-slot:activator="{ on, attrs }">
            <v-btn icon v-on="on" v-bind="attrs">
              <v-icon>mdi-dots-vertical</v-icon>
            </v-btn>
          </template>
          <v-list>
            <v-list-item :href="`${env && env.apiUrl}/guild-rss/${guild.id}`" target="_blank">
              <v-list-item-icon>
                <v-icon dark>mdi-rss</v-icon>
              </v-list-item-icon>
              <v-list-item-content>
                RSS
              </v-list-item-content>
            </v-list-item>
            <v-list-item v-if="storeAccount.apiKey" :href="`${env && env.apiUrl}/patron-api/games?key=${storeAccount.apiKey}&guildId=${guild.id}`" target="_blank">
              <v-list-item-icon>
                <v-icon dark>mdi-key-variant</v-icon>
              </v-list-item-icon>
              <v-list-item-content>
                API
              </v-list-item-content>
            </v-list-item>
          </v-list>
        </v-menu>
      </v-toolbar>

      <v-container fluid v-if="!guild.collapsed">
        <v-row dense>
          <v-col
            v-for="(game, i) in guild.games.filter(game => !game.deleted && !game.filtered)"
            v-bind:key="i"
            cols="12"
            sm="6"
            md="4"
            lg="4"
            xl="2"
          >
            <GameCard :gameData="game" :numColumns="1" :exclude="['server']"></GameCard>
          </v-col>
        </v-row>
      </v-container>
    </v-card>

    <v-btn
      fab
      :to="config.urls.game.create.path"
      fixed
      right
      bottom
      color="discord"
      :title="lang.buttons && lang.buttons.NEW_GAME"
      style="bottom: 15px;"
      class="hidden-lg-and-up"
      v-if="guilds.find(guild => (guild.permission || guild.isAdmin) && guild.announcementChannels.length > 0)"
    >
      <v-icon>mdi-plus</v-icon>
    </v-btn>
  </v-container>
</template>

<script>
import GameCard from "../../../components/game-card";
import { cloneDeep } from "lodash";

export default {
  middleware: ["authenticated"],
  head: {
    title: "Upcoming Games"
  },
  components: {
    GameCard: GameCard
  },
  data() {
    return {
      guilds: [],
      lang: {},
      rsvpGameId: 0,
      config: this.$store.getters.config,
      env: this.$store.getters.env,
      searchQuery: this.$route.query.s
    };
  },
  computed: {
    storeAccount() {
      return this.$store.getters.account;
    },
    storeGuilds() {
      return this.$store.getters.account
        ? this.$store.getters.account.guilds
        : [];
    },
    storeLang() {
      return this.$store.getters.lang;
    }
  },
  watch: {
    storeGuilds: {
      handler: function(newVal) {
        this.guilds = cloneDeep(newVal).map(g => ({
          ...g,
          games: g.games.filter(game => {
            return game.timestamp >= new Date().getTime();
          }),
          collapsed: false
        }));
        this.searchGuild();
      },
      immediate: true
    },
    storeLang: {
      handler: function(newVal) {
        if (newVal && newVal.nav) this.lang = newVal;
      },
      immediate: true
    }
  },
  mounted() {
    this.$store.commit("setLastListingPage", 'upcoming');
  },
  methods: {
    collapseAll() {
      this.guilds = this.guilds.map(g => {
        g.collapsed = true;
        return g;
      });
    },
    expandAll() {
      this.guilds = this.guilds.map(g => {
        g.collapsed = false;
        return g;
      });
    },
    search($event) {
      if (!$event || $event.key != "Enter") return;
      if (this.searchQuery.trim().length == 0) {
        this.$router.push(this.$route.path);
      } else {
        this.$router.push(
          `${this.$route.path}?s=${encodeURIComponent(this.searchQuery.trim())}`
        );
      }
      this.searchGuild();
    },
    searchGuild() {
      // Regex Example: https://regex101.com/r/LFEUgY/2
      // Removed lookback for lack of Firefox support
      const regex = /((\w+):)?"([^"]+)"|((\w+):)?([^ ]+)/gm,
        matches = [];
      let m;
      if (this.searchQuery) {
        while ((m = regex.exec(this.searchQuery)) !== null) {
          if (m.index === regex.lastIndex) {
            regex.lastIndex++;
          }
          if (m[3] && m[3].length > 0) {
            matches.push({ type: m[2] || "any", query: m[3] });
          }
          if (m[6] && m[6].length > 0) {
            matches.push({ type: m[5] || "any", query: m[6] });
          }
        }
      }
      this.guilds = this.guilds.map(guild => {
        guild.games = guild.games.map(game => {
          if (matches.length > 0) {
            if (
              matches
                .map(match => ({
                  type: match.type,
                  query: match.query,
                  regex: new RegExp(match.query, "gi")
                }))
                .filter(match => {
                  return (
                    (match.query === "new" &&
                      new Date().getTime() - game.createdTimestamp <
                        24 * 3600 * 1000) ||
                    (match.query != "new" &&
                      match.type === "any" &&
                      (match.regex.test(game.adventure) ||
                        match.regex.test(game.dm.tag || game.dm) ||
                        match.regex.test(game.author && game.author.tag) ||
                        match.regex.test(guild.name))) ||
                    match.regex.test(
                      (match.type === "gm" && (game.dm.tag || game.dm)) ||
                        (match.type === "author" &&
                          game.author &&
                          game.author.tag) ||
                        (match.type === "name" && game.adventure) ||
                        (match.type === "server" && guild.name) ||
                        (match.type === "reserved" &&
                          game.reserved.reduce(
                            (i, r) => `${i}\n${r.tag}`,
                            ""
                          )) ||
                        game[match.type]
                    )
                  );
                }).length != matches.length
            ) {
              game.filtered = true;
            } else {
              game.filtered = false;
            }
          } else {
            game.filtered = false;
          }
          return game;
        });
        if (
          !this.searchQuery ||
          this.searchQuery.trim().length === 0 ||
          guild.games.find(game => !game.filtered)
        )
          guild.filtered = false;
        return guild;
      });
      this.expandAll();
    }
  }
};
</script>

<style>
@media (min-width: 1400px) {
  .col-lg-4 {
    flex: 0 0 calc(100% / 4);
    max-width: calc(100% / 4);
  }
}

@media (min-width: 1800px) {
  .col-xl-2 {
    flex: 0 0 calc(100% / 5);
    max-width: calc(100% / 5);
  }
}
</style>