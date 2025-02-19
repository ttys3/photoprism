<template>
    <div class="p-page p-page-albums" v-infinite-scroll="loadMore" :infinite-scroll-disabled="scrollDisabled"
         :infinite-scroll-distance="10" :infinite-scroll-listen-for-event="'scrollRefresh'">

        <v-form ref="form" class="p-albums-search" lazy-validation @submit.prevent="updateQuery" dense>
            <v-toolbar flat color="secondary">
                <v-text-field class="pt-3 pr-3"
                              single-line
                              label="Search"
                              prepend-inner-icon="search"
                              clearable
                              color="secondary-dark"
                              @click:clear="clearQuery"
                              v-model="filter.q"
                              @keyup.enter.native="updateQuery"
                              id="search"
                ></v-text-field>

                <v-spacer></v-spacer>

                <v-btn icon @click.prevent="create">
                    <v-icon>add</v-icon>
                </v-btn>
            </v-toolbar>
        </v-form>

        <v-container fluid class="pa-2">
            <p-scroll-top></p-scroll-top>

            <v-container grid-list-xs fluid class="pa-0 p-albums p-albums-details">
                <v-card v-if="results.length === 0" class="p-albums-empty" flat>
                    <v-card-title primary-title>
                        <div>
                            <h3 class="title mb-3">No albums matched your search</h3>
                            <div>Try again using a different term or
                                <v-btn @click.prevent.stop="create" small>create a new album</v-btn>
                            </div>
                        </div>
                    </v-card-title>
                </v-card>
                <v-layout row wrap class="p-results">
                    <v-flex
                            v-for="(album, index) in results"
                            :key="index"
                            class="p-album"
                            xs6 sm4 md3 lg2 d-flex
                    >
                        <v-hover>
                            <v-card tile class="elevation-0 ma-1 accent lighten-3">
                                <v-img
                                        :src="album.getThumbnailUrl('tile_500')"
                                        aspect-ratio="1"
                                        style="cursor: pointer"
                                        class="accent lighten-2"
                                        @click.prevent="openAlbum(index)"
                                >
                                    <v-layout
                                            slot="placeholder"
                                            fill-height
                                            align-center
                                            justify-center
                                            ma-0
                                    >
                                        <v-progress-circular indeterminate
                                                             color="accent lighten-5"></v-progress-circular>
                                    </v-layout>
                                </v-img>

                                <v-card-actions>
                                    <v-edit-dialog
                                            :return-value.sync="album.AlbumName"
                                            lazy
                                            @save="onSave(album)"
                                            class="p-inline-edit"
                                    >
                                        <span v-if="album.AlbumName">
                                            {{ album.AlbumName }}
                                        </span>
                                        <span v-else>
                                            <v-icon>edit</v-icon>
                                        </span>
                                        <template v-slot:input>
                                            <div class="mt-3 title">Change Title</div>
                                        </template>
                                        <template v-slot:input>
                                            <v-text-field
                                                    v-model="album.AlbumName"
                                                    :rules="[titleRule]"
                                                    label="Title"
                                                    color="secondary-dark"
                                                    single-line
                                                    autofocus
                                            ></v-text-field>
                                        </template>
                                    </v-edit-dialog>

                                    <v-spacer></v-spacer>
                                    <v-btn icon @click.stop.prevent="album.toggleLike()">
                                        <v-icon v-if="album.AlbumFavorite" color="#FFD600">star
                                        </v-icon>
                                        <v-icon v-else color="accent lighten-2">star</v-icon>
                                    </v-btn>
                                </v-card-actions>
                            </v-card>
                        </v-hover>
                    </v-flex>
                </v-layout>
            </v-container>
        </v-container>
    </div>
</template>

<script>
    import Album from "model/album";
    import {DateTime} from "luxon";

    export default {
        name: 'p-page-albums',
        props: {
            staticFilter: Object
        },
        watch: {
            '$route'() {
                const query = this.$route.query;

                this.filter.q = query['q'];
                this.lastFilter = {};
                this.routeName = this.$route.name;
                this.search();
            }
        },
        data() {
            const query = this.$route.query;
            const routeName = this.$route.name;
            const q = query['q'] ? query['q'] : '';
            const filter = {q: q};
            const settings = {};

            return {
                results: [],
                scrollDisabled: true,
                pageSize: 24,
                offset: 0,
                selection: this.$clipboard.selection,
                settings: settings,
                filter: filter,
                lastFilter: {},
                routeName: routeName,
                titleRule: v => v.length <= 25 || "Title too long",
            };
        },
        methods: {
            clearQuery() {
                this.filter.q = '';
                this.search();
            },
            openAlbum(index) {
                const album = this.results[index];
                this.$router.push({name: "album", params: { uuid: album.AlbumUUID, slug: album.AlbumSlug }});
            },
            loadMore() {
                if (this.scrollDisabled) return;

                this.scrollDisabled = true;

                this.offset += this.pageSize;

                const params = {
                    count: this.pageSize,
                    offset: this.offset,
                };

                Object.assign(params, this.lastFilter);

                Album.search(params).then(response => {
                    this.results = this.results.concat(response.models);

                    this.scrollDisabled = (response.models.length < this.pageSize);

                    if (this.scrollDisabled) {
                        this.$notify.info("All " + this.results.length + " albums loaded");
                    }
                });
            },
            updateQuery() {
                const query = {
                    view: this.settings.view
                };

                Object.assign(query, this.filter);

                for (let key in query) {
                    if (query[key] === undefined || !query[key]) {
                        delete query[key];
                    }
                }

                this.$router.replace({query: query});
            },
            searchParams() {
                const params = {
                    count: this.pageSize,
                    offset: this.offset,
                };

                Object.assign(params, this.filter);

                if (this.staticFilter) {
                    Object.assign(params, this.staticFilter);
                }

                return params;
            },
            search() {
                this.scrollDisabled = true;

                // Don't query the same data more than once
                if (JSON.stringify(this.lastFilter) === JSON.stringify(this.filter)) {
                    this.$nextTick(() => this.$emit("scrollRefresh"));
                    return;
                }

                Object.assign(this.lastFilter, this.filter);

                this.offset = 0;

                const params = this.searchParams();

                Album.search(params).then(response => {
                    this.results = response.models;

                    this.scrollDisabled = (response.models.length < this.pageSize);

                    if (this.scrollDisabled) {
                        if (!this.results.length) {
                            this.$notify.warning("No albums found");
                        } else if (this.results.length === 1) {
                            this.$notify.info("One album found");
                        } else {
                            this.$notify.info(this.results.length + " albums found");
                        }
                    } else {
                        this.$notify.info('More than 20 albums found');

                        this.$nextTick(() => this.$emit("scrollRefresh"));
                    }
                });
            },
            refresh() {
                this.lastFilter = {};
                const pageSize = this.pageSize;
                this.pageSize = this.offset + pageSize;
                this.search();
                this.offset = this.pageSize;
                this.pageSize = pageSize;
            },
            create() {
                const name = DateTime.local().toFormat("LLLL yyyy");
                const album = new Album({"AlbumName": name});

                album.save().then(() => {
                    this.filter.q = "";
                    this.lastFilter = {};

                    this.search();
                })
            },
            onSave(album) {
                album.update();
            },
        },
        created() {
            this.search();
        },
    };
</script>
