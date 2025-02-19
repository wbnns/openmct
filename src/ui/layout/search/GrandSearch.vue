/*****************************************************************************
 * Open MCT, Copyright (c) 2014-2022, United States Government
 * as represented by the Administrator of the National Aeronautics and Space
 * Administration. All rights reserved.
 *
 * Open MCT is licensed under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 * http://www.apache.org/licenses/LICENSE-2.0.
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
 * WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
 * License for the specific language governing permissions and limitations
 * under the License.
 *
 * Open MCT includes source code licensed under additional open source
 * licenses. See the Open Source Licenses file (LICENSES.md) included with
 * this source code distribution or the Licensing information page available
 * at runtime from the About dialog for additional information.
 *****************************************************************************/

<template>
<div
    ref="GrandSearch"
    aria-label="OpenMCT Search"
    class="c-gsearch"
    role="searchbox"
>
    <search
        ref="shell-search"
        class="c-gsearch__input"
        tabindex="0"
        :value="searchValue"
        @input="searchEverything"
        @clear="searchEverything"
        @click="showSearchResults"
    />
    <SearchResultsDropDown
        ref="searchResultsDropDown"
    />

</div>
</template>

<script>
import search from '../../components/search.vue';
import SearchResultsDropDown from './SearchResultsDropDown.vue';

export default {
    name: 'GrandSearch',
    components: {
        search,
        SearchResultsDropDown
    },
    inject: ['openmct'],
    props: {
    },
    data() {
        return {
            searchValue: '',
            searchLoading: false,
            annotationSearchResults: [],
            objectSearchResults: []
        };
    },
    destroyed() {
        document.body.removeEventListener('click', this.handleOutsideClick);
    },
    methods: {
        async searchEverything(value) {
            // if an abort controller exists, regardless of the value passed in,
            // there is an active search that should be canceled
            if (this.abortSearchController) {
                this.abortSearchController.abort();
                delete this.abortSearchController;
            }

            this.searchValue = value;
            this.searchLoading = true;
            // clear any previous search results
            this.annotationSearchResults = [];
            this.objectSearchResults = [];

            if (this.searchValue) {
                await this.getSearchResults();
            } else {
                this.searchLoading = false;
                this.$refs.searchResultsDropDown.showResults(this.annotationSearchResults, this.objectSearchResults);
            }
        },
        getPathsForObjects(objectsNeedingPaths) {
            return Promise.all(objectsNeedingPaths.map(async (domainObject) => {
                const keyStringForObject = this.openmct.objects.makeKeyString(domainObject.identifier);
                const originalPathObjects = await this.openmct.objects.getOriginalPath(keyStringForObject);

                return {
                    originalPath: originalPathObjects,
                    ...domainObject
                };
            }));
        },
        async getSearchResults() {
            // an abort controller will be passed in that will be used
            // to cancel an active searches if necessary
            this.abortSearchController = new AbortController();
            const abortSignal = this.abortSearchController.signal;
            try {
                this.annotationSearchResults = await this.openmct.annotation.searchForTags(this.searchValue, abortSignal);
                const fullObjectSearchResults = await Promise.all(this.openmct.objects.search(this.searchValue, abortSignal));
                const aggregatedObjectSearchResults = fullObjectSearchResults.flat();
                const aggregatedObjectSearchResultsWithPaths = await this.getPathsForObjects(aggregatedObjectSearchResults);
                const filterAnnotations = aggregatedObjectSearchResultsWithPaths.filter(result => {
                    return result.type !== 'annotation';
                });
                this.objectSearchResults = filterAnnotations;
                this.showSearchResults();
            } catch (error) {
                console.error(`😞 Error searching`, error);
                this.searchLoading = false;

                if (this.abortSearchController) {
                    delete this.abortSearchController;
                }
            }
        },
        showSearchResults() {
            this.$refs.searchResultsDropDown.showResults(this.annotationSearchResults, this.objectSearchResults);
            document.body.addEventListener('click', this.handleOutsideClick);
        },
        handleOutsideClick(event) {
            // if click event is detected outside the dropdown while the
            // dropdown is visible, this will collapse the dropdown.
            if (this.$refs.GrandSearch) {
                const clickedInsideDropdown = this.$refs.GrandSearch.contains(event.target);
                const clickedPreviewClose = event.target.parentElement && event.target.parentElement.querySelector('.js-preview-window');
                const searchResultsDropDown = this.$refs.searchResultsDropDown._data;
                if (!clickedInsideDropdown && searchResultsDropDown.resultsShown && !searchResultsDropDown.previewVisible && !clickedPreviewClose) {
                    searchResultsDropDown.resultsShown = false;
                    document.body.removeEventListener('click', this.handleOutsideClick);
                }
            }
        }
    }
};
</script>
