<script lang="ts">
import { defineComponent } from 'vue'
import { useQuasar } from 'quasar'
import api from 'axios'

type TextError = {
	index: number,
	offset: number,
	sentence: string,
	message: string,
	replacements: {label: string; value: string}[],
	type: {typeName: string},
	rule: {id: string},
	length: number
}

type TextErrors = [TextError]

export default defineComponent({
    name: 'IndexPage',
    data() {
        return {
            textArea: '' as string,
            textAreaSave: '' as string,
            readOnly: false as boolean,
            wordsCount: 0 as number,
            showDialog: false as boolean,
            showNoError: false as boolean,
            sentenceRanges: [] as [number, number][],
            indexTextErrors: [] as number[],
            textErrors: [] as object[], 
            textError: {
                index: 0 as number,
                offset: 0 as number,
                difference: 0 as number,
                message: '' as string,
                sentence: '' as string,
                sentenceCopy: '' as string,
                sentenceIndex: 0 as number,
                replacements: [] as {label: string, value: string}[], 
                fault: '' as string,
                faultLength: 0 as number,
                correction: '' as string,
            },
            lineBreakVerified: false as boolean,
            lineBreaksSentence: [] as [number, number][],
            // Explication pour lineBreaksSentence: 
            // Lors de la requête, la backend ne prends pas en compte les retours à la ligne dans le texte, elle les enlève tous
            // Donc dans l'appel à l'API on transforme les retours à la ligne en un seul espace.
            charOverwrite: false as boolean
        }
    },
    setup() {
        const $q = useQuasar();
        
        return {
            showLoading() {
                $q.loading.show({delay: 300, message: 'Vérification en cours. Veuillez patienter...'});
            }, 
            hideLoading() {
                $q.loading.hide();
            }
        }
    },
    methods: {
        copyText() {
            const textArea = this.textArea;
            
            try {
                navigator.clipboard.writeText(textArea);
            }
            catch(error) {
                console.error('Erreur: ', error);
            }
        },
        getWordsCount() {
            if (this.textArea.length !== 0) {
                //this.wordsCount = this.textArea.trim().split(/\s+/).length;
                this.wordsCount = this.textArea.replace(/[,.?!]/g, '').trim().split(/\s+/).length
            }
            else {
                this.wordsCount = 0;
            }
        },
        removeWhiteSpace() {
            // Retire les espaces inutiles dans la textArea (après une phrase par exemple)
            this.textArea = this.textArea.replace(/[^\S\r\n]+$/gm, '')
        },
        setLineBreaks() {
            // Si il n'y a pas eu de vérification de sauts de lignes dans le texte
            if (!this.lineBreakVerified) {
                this.lineBreaksSentence = []
                var indexSentence = 0
                var tab = []

                // Création des emplacements de sauts de lignes en fonction du nombre de phrases
                for (var i = 0; i != this.sentenceRanges.length; i++) {
                    this.lineBreaksSentence.push([]);
                }

                // Parcours tout le texte et toutes les phrases
                for (var i = 0; i != this.textArea.length - 1 && indexSentence != this.lineBreaksSentence.length; i++) {
                    // Si i est compris dans la phrase actuelle
                    if (i >= this.sentenceRanges[indexSentence][0] && i <= this.sentenceRanges[indexSentence][1]) {
                        // Si il y au minimum 2 retours à la ligne à la suite
                        if ((this.textArea[i-1] == '\n' && this.textArea[i] == '\n') || (this.textArea[i] == '\n' && this.textArea[i+1] == '\n')) {
                            // On ajoute l'emplacement dans tab
                            tab.push(i)
                            // On ajoute l'emplacement du saut de ligne
                            this.lineBreaksSentence[indexSentence].push(i)
                        }
                        else {
                            if (tab.length != 0) {
                                this.setSentenceRanges(indexSentence)
                            }
                            tab = []
                        }
                    }
                    else {
                        // Passage à l'index de l'autre phrase
                        indexSentence++
                        i--
                    }
                }
            }
            this.lineBreakVerified = true
        },
        hasAtleastTwoLineBreaksBefore() {
            var nb = 0
            var lastLineBreakSentence = 0

            // Parcours la liste des sauts de lignes
            for (var i = 0; i != this.lineBreaksSentence.length; i++) {
                for (var j = 0; j != this.lineBreaksSentence[i].length; j++) {
                    // Si l'offset est plus grand que l'index du saut de ligne
                    if (this.textError.offset + this.textError.difference + nb >= this.lineBreaksSentence[i][j]) {
                        // Si le dernier index de saut de ligne de la phrase précédente suit le premier index de la phrase actuelle
                        if (lastLineBreakSentence != 0 && (lastLineBreakSentence+1) == this.lineBreaksSentence[i][j]) {
                            nb++
                        }
                        
                        // Si l'index du saut de ligne actuel est suivi par le prochain saut de ligne
                        if (this.lineBreaksSentence[i][j] == (this.lineBreaksSentence[i][j+1] - 1)) {
                            nb++
                        }
                    }

                    // Si la boucle for (i) va changer de phrase
                    if (j == this.lineBreaksSentence[i].length - 1) {
                        // Sauvegarde du dernier index de la phrase
                        lastLineBreakSentence = this.lineBreaksSentence[i][j]
                    }
                }
            }
            return nb
        },
        setSentenceRanges(sentenceIndex: number) {
            // Recoit l'index de la phrase actuelle
            // Décale toutes les bornes des phrases suivant la phrase actuelle
            for (var i = sentenceIndex; i != this.sentenceRanges.length; i++) {
                if (i == sentenceIndex) {
                    // Ajout uniquement à sa borne de fin
                    this.sentenceRanges[i][1] += 1
                }
                else {
                    this.sentenceRanges[i][0] += 1
                    this.sentenceRanges[i][1] += 1
                }
            }
        },
        editSentenceRanges() {
            // Si la correction est plus grande que la faute
            if (this.textError.correction.length > this.textError.fault.length) {
                const diffTaille = this.textError.correction.length - this.textError.fault.length

                // Parcours de la liste d'index des phrases en partant de celle où se trouve la faute
                for (var i = this.textError.sentenceIndex; i != this.sentenceRanges.length; i++) {
                    // Si la phrase actuelle est celle où se trouve la faute
                    if (i == this.textError.sentenceIndex) {
                        // Ajout de la différence uniquement à sa borne de fin
                        this.sentenceRanges[i][1] += diffTaille
                    }
                    // Sinon
                    else {
                        // Ajout de la différence aux 2 bornes
                        this.sentenceRanges[i][0] += diffTaille
                        this.sentenceRanges[i][1] += diffTaille
                    }
                }

                // Parcours de la liste des sauts de lignes 
                for (var i = this.textError.sentenceIndex; i != this.lineBreaksSentence.length; i++) {
                    for (var j = 0; j != this.lineBreaksSentence[i].length; j++) {
                        // Si les sauts de lignes sont après la faute
                        if (this.textError.offset < this.lineBreaksSentence[i][j]) {
                            this.lineBreaksSentence[i][j] += diffTaille
                        }
                    }
                }
            }

            // Si la correction est plus petite que la faute
            else if (this.textError.correction.length < this.textError.fault.length) {
                const diffTaille = this.textError.fault.length - this.textError.correction.length

                // Parcours de la liste d'index des phrases en partant de celle où se trouve la faute
                for (var i = this.textError.sentenceIndex; i != this.sentenceRanges.length; i++) {
                    // Si la phrase actuelle est celle où se trouve la faute
                    if (i == this.textError.sentenceIndex) {
                        // Diminution de la différence uniquement à sa borne de fin
                        this.sentenceRanges[i][1] -= diffTaille
                    }
                    // Sinon
                    else {
                        // Diminution de la différence aux 2 bornes
                        this.sentenceRanges[i][0] -= diffTaille
                        this.sentenceRanges[i][1] -= diffTaille
                    }
                }

                // Parcours de la liste des sauts de lignes 
                for (var i = this.textError.sentenceIndex; i != this.lineBreaksSentence.length; i++) {
                    for (var j = 0; j != this.lineBreaksSentence[i].length; j++) {
                        // Si les sauts de lignes sont après la faute
                        if (this.textError.offset < this.lineBreaksSentence[i][j]) {
                            this.lineBreaksSentence[i][j] -= diffTaille
                        }
                    }
                }
            }
        },
        async checkText() {
            // Si il y a bien un texte
            if (this.textArea.length !== 0) {
                this.showLoading();

                this.textAreaSave = this.textArea.replace(/\s{2,}/g,' ')
                
                try {
                    this.removeWhiteSpace()

                    const JSON = await api.post('http://localhost:8081/v2/check?language=fr&text='+ this.textArea.replace(/\s+/g, ' '));
                    
                    // Définition des bornes de fin et de début des phrases
                    this.sentenceRanges = JSON.data.sentenceRanges;
                    
                    // Définition des fautes dans le texte
                    this.textErrors = JSON.data.matches as TextErrors;

                    // Reset du tableau d'index de fautes
                    this.indexTextErrors = []

                    // Si aucune faute
                    if (this.textErrors.length == 0) {
                        this.hideLoading()

                        // Affiche la pop-up: aucune faute détectée
                        this.showNoError = true
                    }
                    else {
                        this.hideLoading()
                        
                        // Reset de la différence
                        this.textError.difference = 0

                        // Reset du booléen de la vérification des sauts de lignes
                        this.lineBreakVerified = false

                        // Désactivation de l'input dans la textArea
                        this.readOnly = true

                        this.setLineBreaks();
                        this.updateTextErrorColor('resetAll');
                        
                        this.hideLoading();
                    }
                }
                catch (error) {
                    console.error('Erreur:', error)
                    this.hideLoading();
                }
            }
        },
        setTextError(index: number) {
            this.textError.index = index; 
            this.textError.offset = (this.textErrors[this.textError.index] as TextError).offset;
            this.textError.message = (this.textErrors[this.textError.index] as TextError).message;
            // Si sentence n'a jamais été initialisée alors on init / Sinon on garde la sentence actuelle
            this.textError.sentence = this.textError.sentence == '' ? (this.textErrors[this.textError.index] as TextError).sentence : this.textError.sentence; 
            this.textError.sentenceCopy = this.textError.sentence
            this.textError.replacements = (this.textErrors[this.textError.index] as TextError).replacements;
            this.textError.fault = '';
            this.textError.faultLength = (this.textErrors[this.textError.index] as TextError).length;
        },
        showTextError(index: number) {
            this.setTextError(index)
            
            // Parcours des phrases
            for (var i = 0; i != this.sentenceRanges.length; i++) {
                // Si on a changé de phrase
                if ((this.textError.offset + this.textError.difference >= this.sentenceRanges[i][0]) && (this.textError.offset + this.textError.difference <= this.sentenceRanges[i][1]) && this.textError.sentenceIndex != i) {
                    // Maj de l'index de la phrase par rapport à SentenceRanges
                    this.textError.sentenceIndex = i;
                    
                    // Construction de la nouvelle phrase à partir de textArea
                    var sentence = ''
                    
                    for (var j = this.sentenceRanges[this.textError.sentenceIndex][0]; j < this.sentenceRanges[this.textError.sentenceIndex][1]; j++) {
                        sentence += this.textArea[j]
                    }

                    // Peut pas enlever les espaces car après changement de phrase l'offset se sert de sentence
                    //sentence = sentence.replace(/\n/g, '')
                    this.textError.sentence = sentence
                    this.textError.sentenceCopy = this.textError.sentence
                }
            }

            // Crée une clé label pour chaque correction pour pouvoir afficher les valeurs dans le select
            for (var i = 0; i != this.textError.replacements.length; i++) {
                this.textError.replacements[i].label = this.textError.replacements[i].value;
            }

            // Si il y a au moins une correction proposée
            if (this.textError.replacements.length != 0) {
                // Applique la 1ère valeur du select par défaut
                this.textError.correction = this.textError.replacements[0].label;
            }

            // Si il y a eu écrasement de caractère avant
            if (this.charOverwrite) {
                this.textError.difference -= 1

                // Calcul du nouveau offset
                this.textError.offset += this.textError.difference + this.hasAtleastTwoLineBreaksBefore()
                
                this.charOverwrite = false
            }
            // Sinon
            else {
                // Calcul du nouveau offset
                this.textError.offset += this.textError.difference + this.hasAtleastTwoLineBreaksBefore()
            }

            // Construction du mot à corriger
            for (var i = this.textError.offset; i != this.textError.offset + this.textError.faultLength; i++) {
                this.textError.fault += this.textArea[i];
            }

            // Affichage de la popup
            this.showDialog = true;
        },
        applyCorrection() {
            // Si l'index de l'erreur corrigée n'existe pas
            if (!this.indexTextErrors.find((index) => index == this.textError.index)) {
                // On ajoute l'index de l'erreur corrigée
                this.indexTextErrors.push(this.textError.index)
            }
            
            var newSentence = ''

            // Ajout du découpage du début de la phrase jusqu'à la faute
            newSentence += this.textArea.slice(this.sentenceRanges[this.textError.sentenceIndex][0], this.textError.offset)

            // Ajout de la correction
            newSentence += this.textError.correction

            // Ajout du découpage du reste de la phrase
            newSentence += this.textArea.slice(this.textError.offset + this.textError.faultLength, this.sentenceRanges[this.textError.sentenceIndex][1])

            // Mise à jour de la phrase
            this.textError.sentence = newSentence
            
            if (this.textError.correction != '') {
                // Si la faute est plus longue que la correction
                if (this.textError.fault.length > this.textError.correction.length) {
                    // On calcule la différence de longueur
                    this.textError.difference -= (this.textError.fault.length - this.textError.correction.length)
                }
                // Sinon si la faute est plus petite que la correction
                else if (this.textError.fault.length < this.textError.correction.length) {
                    // On calcule la différence de longueur
                    this.textError.difference += (this.textError.correction.length - this.textError.fault.length)
                }
            }
            else {
                this.charOverwrite = true
            }

            // Modif des index de début et de fin des phrases
            this.editSentenceRanges()

            var texte = ''

            // Ajout du découpage du début du texte jusqu'à la faute
            texte += this.textArea.slice(0, this.textError.offset)

            // Ajout de la correction
            texte += this.textError.correction

            // Ajout du découpage du reste du texte 
            texte += this.textArea.slice(this.textError.offset + this.textError.faultLength)

            // Mise à jour de la textArea
            this.textArea = texte

            this.updateTextErrorColor('set');

            // Si toutes les erreurs ont été corrigées
            if (this.indexTextErrors.length == this.textErrors.length) {
                // On enlève le read-only
                this.readOnly = false
            }
        },
        updateTextErrorColor(mode: string) {
            if (mode === 'set') {
                const textError = document.getElementById('fault-' + (this.textError.index+1));

                textError?.classList.remove('bg-red-10');
                textError?.classList.add('bg-positive');
                textError?.classList.add('disabled');
                textError?.setAttribute('disabled', 'true');
            }
            
            else if (mode === 'resetAll') {
                for (var i = 0; i != this.textErrors.length; i++) {
                    const textError = document.getElementById('fault-' + (i+1));

                    textError?.classList.remove('bg-positive');
                    textError?.classList.add('bg-red-10');
                    textError?.removeAttribute('disabled');
                    textError?.classList.remove('disabled');
                }
            } 
        }
    }
})
</script>

<template>
    <q-dialog v-model="showDialog">
        <q-card style="width: 750px; max-width: none;" align="center">

            <q-card-section>
                <div class="text-h4">Faute n°{{ (textError.index + 1) }}</div>
            </q-card-section>

            <q-card-section class="text-body1">
                <span>{{ textError.message }}</span>
                <span v-if="textError.replacements.length != 0"> Veuillez choisir une correction parmi la liste.</span><br><br>
                <span>"{{ textError.sentence }}"</span>
            </q-card-section>

            <q-card-actions v-if="textError.replacements.length != 0" align="center" >
                <q-select v-model="textError.correction" :options="textError.replacements" emit-value fill-input class="text-body1" filled />
                <q-btn @click="applyCorrection()" label="Valider" icon="check" color="positive" style="margin-left: 15px; height: 35px;" rounded />
            </q-card-actions>

        </q-card>
    </q-dialog>

    <q-dialog v-model="showNoError">
		<q-card style="width: 750px; max-width: none" align="center">

			<q-card-section>
				<div class="text-h4">Aucune faute trouvée</div>
			</q-card-section>

			<div class="main-check-container">
				<div class="check-container">
					<div class="check-background">
						<svg viewBox="0 0 65 51" fill="none">
							<path d="M7 25L27.3077 44L58.5 7" stroke="white" stroke-width="13" stroke-linecap="round" stroke-linejoin="round" />
						</svg>
					</div>
					<div class="check-shadow"></div>
				</div>
			</div>

		</q-card>
	</q-dialog>

    <q-page class="flex flex-center">

        <div id="main" class="shadow-6">
            <div id="top">
                <q-btn @click="copyText()" icon="mdi-content-copy" class="q-mr-md"> 
                    <q-tooltip anchor="top middle" self="bottom middle">
                        Copier
                    </q-tooltip>
                </q-btn>
            </div>

            <q-input type="textarea" v-model="textArea" @change="getWordsCount()" :readonly="readOnly" spellcheck="false" id="textArea" ref="textArea" class="text-h5" filled />
            
            <div id="bottom" class="flex justify-between">
                <span class="q-pt-sm">Mots: {{ wordsCount }} </span>
                <q-btn @click="checkText()" label="Vérifier" icon="check" color="primary" />
            </div>
        </div>

        <div id="faultsPanel" class="shadow-6">
            <span id="fault">Fautes :</span>

            <q-btn v-for="(fault, index) in textErrors" :key="index" :id="'fault-'+(index+1)" @click="showTextError(index)" class="faults" color="red-10" >
                <span class="faultTitle">Faute n°{{ (index + 1) }}</span><br>
                <span>{{ (fault as TextError).message }}</span>
            </q-btn>
        </div>

    </q-page>
</template>


  
